# 知识库数据预处理管道：PDF/Word/HTML 到 Markdown 的工程实践，数据清洗做好了 RAG 效果提升 50%

> 很多人以为 RAG 的核心是向量检索和 LLM，但我告诉你：**如果说 RAG 是一个人的身体，那数据预处理就是骨骼。骨架歪了，再怎么训练肌肉都没用。**

---

## 一、开篇：一个真实的翻车案例

先讲一个我亲身经历的事。

去年给某金融客户做智能合同审查系统。Demo 演示完美，20 份合同文档，问答准确率 95%+。老板当场拍板："下周上生产，先导入过去 3 年的 5 万份合同。"

结果刚导了 3000 份，QA 就跑过来："这 AI 给出的答案怎么有乱码？"

排查了半天，问题出在一份 2007 年的扫描版 PDF 合同上——OCR 识别把"甲方应支付人民币壹佰万元"识别成了"甲方应支付人民巾壹伯万元"。"币"变成了"巾"，"佰"变成了"伯"。

更离谱的是，另一份带复杂表格的合同，Apache PDFBox 把表格拆成了碎片，导致"违约金为合同总额的20%"这个关键条款丢失了——AI 直接回答"合同中未约定违约金"。

**血的教训：数据预处理不是可有可无的降级项，而是 RAG 系统中决定生死的关键环节。**

这篇文章将从零开始，手把手带你构建一个生产级的 Java 文档预处理管道。不玩虚的，全是代码。

---

## 二、预处理管道整体设计

### 2.1 为什么选择责任链模式

预处理涉及多个步骤：格式识别 → 内容提取 → 清洗 → 分块 → 元数据提取。每个步骤之间相对独立，但又需要按序执行。责任链模式天然适合：

```java
public interface DocumentProcessor {
    
    /**
     * 处理文档
     * @param context 文档处理上下文
     * @return 处理后的上下文
     */
    DocumentContext process(DocumentContext context);
    
    /**
     * 处理器优先级，越小越先执行
     */
    default int getOrder() {
        return 0;
    }
    
    /**
     * 是否可以处理该类型的文档
     */
    default boolean supports(DocumentContext context) {
        return true;
    }
}
```

### 2.2 管道实现

```java
@Component
public class DocumentProcessingPipeline {
    
    private final List<DocumentProcessor> processors;
    
    public DocumentProcessingPipeline(List<DocumentProcessor> processors) {
        // 按 order 排序，保证执行顺序
        this.processors = processors.stream()
            .sorted(Comparator.comparingInt(DocumentProcessor::getOrder))
            .toList();
    }
    
    public ProcessedDocument execute(RawDocument rawDocument) {
        DocumentContext context = DocumentContext.builder()
            .rawDocument(rawDocument)
            .metadata(new HashMap<>())
            .errors(new ArrayList<>())
            .build();
        
        for (DocumentProcessor processor : processors) {
            if (!processor.supports(context)) {
                continue;
            }
            try {
                context = processor.process(context);
                log.debug("Processor [{}] completed for document: {}", 
                    processor.getClass().getSimpleName(), rawDocument.getFileName());
            } catch (Exception e) {
                log.error("Processor [{}] failed for document: {}", 
                    processor.getClass().getSimpleName(), rawDocument.getFileName(), e);
                context.getErrors().add(new ProcessingError(
                    processor.getClass().getSimpleName(), e.getMessage()));
                
                // 根据策略决定是否继续
                if (context.isFailFast()) {
                    throw new ProcessingException("Pipeline aborted", e);
                }
            }
        }
        
        return buildResult(context);
    }
}
```

### 2.3 处理器注册配置

```java
@Configuration
public class PipelineConfiguration {
    
    @Bean
    @Order(1)
    public DocumentProcessor formatDetector() {
        return new FormatDetector();
    }
    
    @Bean
    @Order(2)
    public DocumentProcessor contentExtractor() {
        return new ContentExtractor();
    }
    
    @Bean
    @Order(3)
    public DocumentProcessor headerFooterRemover() {
        return new HeaderFooterRemover();
    }
    
    @Bean
    @Order(4)
    public DocumentProcessor tableExtractor() {
        return new TableExtractor();
    }
    
    @Bean
    @Order(5)
    public DocumentProcessor imageProcessor() {
        return new ImageProcessor();
    }
    
    @Bean
    @Order(6)
    public DocumentProcessor markdownConverter() {
        return new MarkdownConverter();
    }
    
    @Bean
    @Order(7)
    public DocumentProcessor metadataExtractor() {
        return new MetadataExtractor();
    }
    
    @Bean
    @Order(8)
    public DocumentProcessor qualityValidator() {
        return new QualityValidator();
    }
}
```

---

## 三、多格式文档解析实战

### 3.1 PDF 解析：三大工具各显神通

PDF 是 RAG 中最常见的格式，也是最难处理的。没有之一。

```java
@Component
public class ContentExtractor implements DocumentProcessor {
    
    @Override
    public DocumentContext process(DocumentContext context) {
        String fileType = context.getMetadata().get("fileType");
        
        return switch (fileType.toUpperCase()) {
            case "PDF" -> extractPDF(context);
            case "DOCX" -> extractDocx(context);
            case "DOC" -> extractDoc(context);
            case "HTML", "HTM" -> extractHTML(context);
            case "MD", "MARKDOWN" -> extractMarkdown(context);
            case "TXT" -> extractText(context);
            default -> context;
        };
    }
```

**策略一：Apache Tika（万能解药，但性能差）**

```java
private String extractWithTika(InputStream inputStream) throws Exception {
    Parser parser = new AutoDetectParser();
    BodyContentHandler handler = new BodyContentHandler(-1); // -1 禁用长度限制
    Metadata metadata = new Metadata();
    ParseContext parseContext = new ParseContext();
    
    parser.parse(inputStream, handler, metadata, parseContext);
    return handler.toString();
}
```

**策略二：Apache PDFBox（文本型 PDF 首选）**

```java
private String extractWithPDFBox(InputStream inputStream) throws IOException {
    try (PDDocument document = PDDocument.load(inputStream)) {
        PDFTextStripper stripper = new PDFTextStripper();
        
        // 关键配置：保持段落结构
        stripper.setSortByPosition(true);           // 按位置排序
        stripper.setShouldSeparateByBeads(true);    // 按阅读顺序
        stripper.setAddMoreFormatting(true);        // 保留换行
        stripper.setParagraphStart("\n\n");
        stripper.setPageStart("\n--- 第 ");
        stripper.setPageEnd(" 页 ---\n");
        
        String text = stripper.getText(document);
        
        // 提取表格（PDFBox 原生不支持表格，需要这个技巧）
        List<TableInfo> tables = extractTablesFromPDF(document);
        text = injectTableMarkdown(text, tables);
        
        return text;
    }
}
```

**策略三：OCR（扫描件 PDF 的唯一解）**

```java
private String extractWithOCR(InputStream inputStream) throws Exception {
    try (PDDocument document = PDDocument.load(inputStream)) {
        PDFRenderer renderer = new PDFRenderer(document);
        StringBuilder result = new StringBuilder();
        
        for (int page = 0; page < document.getNumberOfPages(); page++) {
            // 将 PDF 页面渲染为图片
            BufferedImage image = renderer.renderImageWithDPI(page, 300); // 300 DPI
            
            // 图片预处理：提高 OCR 准确率
            BufferedImage processed = preprocessForOCR(image);
            
            // Tesseract OCR 识别
            String pageText = doOCR(processed);
            result.append(pageText).append("\n\n--- 第 ")
                  .append(page + 1).append(" 页 ---\n\n");
        }
        return result.toString();
    }
}

private BufferedImage preprocessForOCR(BufferedImage image) {
    // 灰度化
    BufferedImage gray = new BufferedImage(
        image.getWidth(), image.getHeight(), BufferedImage.TYPE_BYTE_GRAY);
    Graphics2D g = gray.createGraphics();
    g.drawImage(image, 0, 0, null);
    g.dispose();
    
    // 二值化（自适应阈值）
    // 去噪（中值滤波）
    return applyAdaptiveThreshold(gray);
}

private String doOCR(BufferedImage image) {
    ITesseract tesseract = new Tesseract();
    tesseract.setLanguage("chi_sim+eng");  // 中英混合
    tesseract.setPageSegMode(6);           // 假设为统一文本块
    tesseract.setOcrEngineMode(1);         // LSTM 模式
    
    try {
        return tesseract.doOCR(image);
    } catch (TesseractException e) {
        throw new RuntimeException("OCR failed", e);
    }
}
```

**智能选择策略：**

```java
private String extractPDF(DocumentContext context) {
    try {
        // 策略1：先用 PDFBox 试（最快）
        String text = extractWithPDFBox(context.getInputStream());
        
        // 如果提取的文字太少（< 每页 50 字符），很可能是扫描件
        int pageCount = getPageCount(context.getInputStream());
        if (text.length() < pageCount * 50) {
            log.warn("PDF appears to be scanned, switching to OCR. Pages: {}", pageCount);
            context.getInputStream().reset();
            return extractWithOCR(context.getInputStream());
        }
        
        return text;
        
    } catch (Exception e) {
        log.warn("PDFBox extraction failed, falling back to Apache Tika");
        try {
            context.getInputStream().reset();
            return extractWithTika(context.getInputStream());
        } catch (Exception ex) {
            throw new ProcessingException("All PDF extraction methods failed", ex);
        }
    }
}
```

### 3.2 Word 文档解析

```java
private String extractDocx(InputStream inputStream) throws Exception {
    try (XWPFDocument document = new XWPFDocument(inputStream)) {
        StringBuilder markdown = new StringBuilder();
        
        for (IBodyElement element : document.getBodyElements()) {
            switch (element.getElementType()) {
                case PARAGRAPH -> {
                    XWPFParagraph paragraph = (XWPFParagraph) element;
                    String style = paragraph.getStyle();
                    
                    if (style != null) {
                        String text = paragraph.getText().trim();
                        if (text.isEmpty()) continue;
                        
                        // 根据 Word 样式转换为 Markdown 标题
                        if (style.startsWith("Heading")) {
                            int level = Integer.parseInt(style.replace("Heading", ""));
                            markdown.append("#".repeat(level)).append(" ").append(text).append("\n\n");
                        }
                        // 检测代码块（等宽字体段落）
                        else if (isCodeBlock(paragraph)) {
                            markdown.append("```\n").append(text).append("\n```\n\n");
                        }
                        // 列表项
                        else if (paragraph.getNumFmt() != null) {
                            markdown.append("- ").append(text).append("\n");
                        } else {
                            markdown.append(text).append("\n\n");
                        }
                    }
                }
                case TABLE -> {
                    XWPFTable table = (XWPFTable) element;
                    markdown.append(convertTableToMarkdown(table)).append("\n\n");
                }
            }
        }
        return markdown.toString();
    }
}

private String convertTableToMarkdown(XWPFTable table) {
    StringBuilder md = new StringBuilder();
    List<XWPFTableRow> rows = table.getRows();
    
    for (int i = 0; i < rows.size(); i++) {
        XWPFTableRow row = rows.get(i);
        md.append("|");
        for (XWPFTableCell cell : row.getTableCells()) {
            md.append(" ").append(cell.getText().replace("\n", " ")).append(" |");
        }
        md.append("\n");
        
        // 表头后添加分隔线
        if (i == 0) {
            md.append("|");
            for (int j = 0; j < row.getTableCells().size(); j++) {
                md.append(" --- |");
            }
            md.append("\n");
        }
    }
    return md.toString();
}
```

### 3.3 HTML 解析

```java
private String extractHTML(InputStream inputStream) throws Exception {
    String html = new String(inputStream.readAllBytes(), StandardCharsets.UTF_8);
    Document document = Jsoup.parse(html);
    
    // 移除无用标签
    document.select("script, style, nav, footer, iframe, noscript, .advertisement, .sidebar").remove();
    
    // 保留标题层级
    for (int i = 1; i <= 6; i++) {
        for (Element heading : document.select("h" + i)) {
            heading.prependText("#".repeat(i) + " ");
        }
    }
    
    // 将 <pre><code> 转为 markdown 代码块
    for (Element codeBlock : document.select("pre code")) {
        codeBlock.parent().text("```\n" + codeBlock.text() + "\n```");
    }
    
    // 将 <table> 转为 markdown 表格
    for (Element table : document.select("table")) {
        table.replaceWith(convertHTMLTableToMarkdown(table));
    }
    
    // 将 <img> 转为 markdown 图片
    for (Element img : document.select("img")) {
        String src = img.attr("src");
        String alt = img.attr("alt");
        img.replaceWith(new TextNode("![" + alt + "](" + src + ")"));
    }
    
    // 提取纯文本（Jsoup 的 text() 已经做了很好的处理）
    return Jsoup.parse(document.html()).wholeText();
}
```

---

## 四、文字提取质量优化

### 4.1 页眉页脚去除

```java
@Component
public class HeaderFooterRemover implements DocumentProcessor {
    
    @Override
    public DocumentContext process(DocumentContext context) {
        String content = context.getContent();
        String[] lines = content.split("\n");
        
        // 策略1：正则匹配常见页眉页脚模式
        Pattern headerPattern = Pattern.compile(
            "^(第\\s*[\\d一二三四五六七八九十百千万]+\\s*页|Page\\s*\\d+|\\d+\\s*/\\s*\\d+)$");
        
        Pattern footerPattern = Pattern.compile(
            "^(机密|Confidential|©|Copyright|版权所有).*$");
        
        Pattern urlPattern = Pattern.compile(
            "^https?://\\S+$");
        
        List<String> cleaned = new ArrayList<>();
        for (String line : lines) {
            String trimmed = line.trim();
            if (trimmed.isEmpty()) {
                cleaned.add(line);
                continue;
            }
            
            // 跳过匹配到的页眉页脚
            if (headerPattern.matcher(trimmed).matches()) continue;
            if (footerPattern.matcher(trimmed).matches()) continue;
            if (urlPattern.matcher(trimmed).matches()) continue;
            
            cleaned.add(line);
        }
        
        context.setContent(String.join("\n", cleaned));
        return context;
    }
}
```

### 4.2 保留层级结构

```java
public class StructurePreserver {
    
    /**
     * 检测并标注文档结构
     * 根据字体大小、加粗、编号等特征推断标题层级
     */
    public String preserveStructure(String content, List<TextBlock> blocks) {
        Deque<Integer> headingStack = new ArrayDeque<>();
        StringBuilder result = new StringBuilder();
        
        for (TextBlock block : blocks) {
            int level = detectHeadingLevel(block);
            
            if (level > 0) {
                // 弹出比当前层级更低的标题
                while (!headingStack.isEmpty() && headingStack.peek() >= level) {
                    headingStack.pop();
                }
                headingStack.push(level);
                result.append("#".repeat(level)).append(" ").append(block.getText()).append("\n\n");
            } else {
                result.append(block.getText()).append("\n\n");
            }
        }
        return result.toString();
    }
    
    private int detectHeadingLevel(TextBlock block) {
        // 基于规则的标题检测
        boolean hasNumbering = Pattern.matches("^[\\d]+[.、]\\s.+", block.getText());
        boolean hasChineseNumbering = Pattern.matches("^[一二三四五六七八九十]+[、.]\\s.+", block.getText());
        boolean isBold = block.isBold();
        boolean isLargerFont = block.getFontSize() > 14;
        boolean isShort = block.getText().length() < 50;
        
        if ((hasNumbering || hasChineseNumbering) && isShort) {
            return hasBoldOrLarger(block) ? 3 : 2;
        }
        
        if (isBold && isLargerFont && isShort) return 1;
        if (isBold && isShort) return 2;
        if (isBold) return 3;
        
        return 0; // 不是标题
    }
}
```

---

## 五、图片与表格的处理策略

### 5.1 图片多模态处理

```java
@Component
public class ImageProcessor implements DocumentProcessor {
    
    private final MultimodalLLMClient multimodalLLM;
    private final MinioClient minioClient;
    
    @Override
    public DocumentContext process(DocumentContext context) {
        List<ImageInfo> images = context.getExtractedImages();
        
        for (ImageInfo image : images) {
            // 步骤1：将图片上传到对象存储
            String imageUrl = uploadToOSS(image.getData(), image.getName());
            image.setUrl(imageUrl);
            
            // 步骤2：多模态 LLM 生成图片描述
            String description = generateImageDescription(image);
            image.setDescription(description);
            
            // 步骤3：OCR 文字提取（如果图片包含文字）
            if (image.isLikelyContainsText()) {
                String ocrText = extractTextFromImage(image.getData());
                image.setOcrText(ocrText);
            }
        }
        
        // 步骤4：将图片描述嵌入到文档内容中
        String contentWithImages = injectImageDescriptions(
            context.getContent(), images);
        context.setContent(contentWithImages);
        
        return context;
    }
    
    private String generateImageDescription(ImageInfo image) {
        String prompt = """
            请详细描述这张图片的内容。如果图片包含数据图表，请尽可能提取其中的关键数据。
            如果图片是流程图或架构图，请描述其结构和各个组件之间的关系。
            
            请用中文回答，控制在 200 字以内。
            """;
        
        return multimodalLLM.describe(new MultimodalRequest(
            image.getUrl(), prompt));
    }
    
    private String injectImageDescriptions(String content, List<ImageInfo> images) {
        String result = content;
        for (ImageInfo image : images) {
            String placeholder = "![%s](%s)".formatted(
                image.getOriginalAlt(), image.getOriginalSrc());
            
            String replacement = """
                [图片描述：%s]
                ![原始图片](%s)
                """.formatted(image.getDescription(), image.getUrl());
            
            result = result.replace(placeholder, replacement);
        }
        return result;
    }
}
```

### 5.2 表格结构保持策略

```java
public class TableExtractor implements DocumentProcessor {
    
    @Override
    public DocumentContext process(DocumentContext context) {
        String content = context.getContent();
        List<TableInfo> tables = detectTables(content);
        
        for (TableInfo table : tables) {
            // 将 ASCII 表格或不规范表格转换为 Markdown 表格
            String markdownTable = convertToMarkdownTable(table);
            content = content.replace(table.getRawText(), markdownTable);
        }
        
        context.setContent(content);
        context.setTables(tables);
        return context;
    }
    
    private List<TableInfo> detectTables(String content) {
        List<TableInfo> tables = new ArrayList<>();
        String[] lines = content.split("\n");
        
        int i = 0;
        while (i < lines.length) {
            // 检测表格开始（连续多行包含 | 或 制表符 或对齐的空白）
            if (looksLikeTableRow(lines[i])) {
                TableInfo table = new TableInfo();
                int start = i;
                
                while (i < lines.length && looksLikeTableRow(lines[i])) {
                    table.addRow(lines[i]);
                    i++;
                }
                
                if (table.getRowCount() >= 2) { // 至少2行才算表格
                    table.setStartLine(start);
                    table.setEndLine(i - 1);
                    tables.add(table);
                }
            } else {
                i++;
            }
        }
        return tables;
    }
    
    private boolean looksLikeTableRow(String line) {
        // | 分隔符
        if (line.trim().contains("|")) return true;
        // 至少2个制表符
        if (line.chars().filter(c -> c == '\t').count() >= 2) return true;
        // 至少3个连续空格分隔的字段
        if (line.trim().split("\\s{3,}").length >= 3) return true;
        return false;
    }
}
```

---

## 六、元数据提取

```java
@Component
public class MetadataExtractor implements DocumentProcessor {
    
    @Override
    public DocumentContext process(DocumentContext context) {
        Map<String, Object> metadata = context.getMetadata();
        String content = context.getContent();
        
        // 基础属性
        metadata.put("fileSize", context.getRawDocument().getFileSize());
        metadata.put("fileType", context.getRawDocument().getFileType());
        metadata.put("fileName", context.getRawDocument().getFileName());
        metadata.put("processedAt", Instant.now().toString());
        metadata.put("contentLength", content.length());
        metadata.put("charCount", content.replaceAll("\\s", "").length());
        
        // 从文件名提取日期（如"2024-Q3-财务报告.pdf"）
        extractDateFromFilename(context);
        
        // 从内容提取标题（第一个 # 标题或前 100 字符中的粗体字）
        extractTitle(context);
        
        // 从内容提取作者
        extractAuthor(context);
        
        // 文档分类（基于内容关键词 + 文件路径）
        classifyDocument(context);
        
        // 计算内容哈希（用于去重）
        metadata.put("contentHash", DigestUtils.md5Hex(content));
        
        // 语言检测
        metadata.put("language", detectLanguage(content));
        
        // 质量评分
        metadata.put("qualityScore", calculateQualityScore(content));
        
        return context;
    }
    
    private void extractTitle(DocumentContext context) {
        String content = context.getContent();
        // 尝试匹配第一个 Markdown 一级标题
        Pattern pattern = Pattern.compile("^#\\s+(.+)$", Pattern.MULTILINE);
        Matcher matcher = pattern.matcher(content);
        
        if (matcher.find()) {
            context.getMetadata().put("title", matcher.group(1).trim());
            return;
        }
        
        // 降级：取第一个非空行
        String firstLine = content.lines()
            .map(String::trim)
            .filter(l -> !l.isEmpty())
            .findFirst()
            .orElse("Untitled");
            
        context.getMetadata().put("title", 
            firstLine.length() > 100 ? firstLine.substring(0, 100) : firstLine);
    }
    
    private void extractAuthor(DocumentContext context) {
        String content = context.getContent();
        
        // 从 Word 元数据
        if ("DOCX".equals(context.getRawDocument().getFileType())) {
            try {
                Tika tika = new Tika();
                org.apache.tika.metadata.Metadata tikaMetadata = new org.apache.tika.metadata.Metadata();
                String author = tikaMetadata.get("Author");
                if (author != null) {
                    context.getMetadata().put("author", author);
                    return;
                }
            } catch (Exception ignored) {}
        }
        
        // 从内容中匹配"作者：XXX"模式
        Pattern pattern = Pattern.compile("(作者|Author|撰写人|编制)[：:]\\s*(.+)");
        Matcher matcher = pattern.matcher(content.substring(0, Math.min(500, content.length())));
        if (matcher.find()) {
            context.getMetadata().put("author", matcher.group(2).trim());
        }
    }
    
    private double calculateQualityScore(String content) {
        double score = 100.0;
        
        // 乱码检测：非正常字符比例
        long garbledChars = content.chars()
            .filter(c -> c == 0xFFFD || c == '�')
            .count();
        if (garbledChars > 10) score -= 30;
        
        // 空白比例过高扣分
        long whitespaceRatio = content.chars()
            .filter(Character::isWhitespace)
            .count() * 100 / Math.max(content.length(), 1);
        if (whitespaceRatio > 70) score -= 20;
        
        // 内容太短扣分
        if (content.length() < 100) score -= 50;
        
        // 全是英文大写（可能是扫描件 OCR 失败）
        long upperCaseRatio = content.chars()
            .filter(Character::isUpperCase)
            .count() * 100 / Math.max(content.length(), 1);
        if (upperCaseRatio > 60) score -= 10;
        
        return Math.max(0, score);
    }
}
```

---

## 七、完整预处理管道调用示例

```java
@Service
@Slf4j
public class DocumentIngestionService {
    
    @Autowired
    private DocumentProcessingPipeline pipeline;
    
    @Autowired
    private EmbeddingService embeddingService;
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private MetadataRepository metadataRepository;
    
    public IngestionResult ingest(MultipartFile file) {
        long startTime = System.currentTimeMillis();
        
        // 1. 构建原始文档对象
        RawDocument raw = RawDocument.builder()
            .fileName(file.getOriginalFilename())
            .fileType(getExtension(file.getOriginalFilename()))
            .fileSize(file.getSize())
            .inputStream(file.getInputStream())
            .build();
        
        // 2. 执行预处理管道
        ProcessedDocument processed = pipeline.execute(raw);
        
        // 3. 检查处理质量
        double qualityScore = (double) processed.getMetadata().get("qualityScore");
        if (qualityScore < 50) {
            log.warn("Low quality document: {}, score: {}", 
                file.getOriginalFilename(), qualityScore);
            // 可以降级处理或人工审核
        }
        
        // 4. 存储元数据
        DocumentMetadataEntity entity = metadataRepository.save(
            DocumentMetadataEntity.builder()
                .title((String) processed.getMetadata().get("title"))
                .author((String) processed.getMetadata().get("author"))
                .fileType(raw.getFileType())
                .fileSize(raw.getFileSize())
                .contentHash((String) processed.getMetadata().get("contentHash"))
                .qualityScore(BigDecimal.valueOf(qualityScore))
                .status("READY")
                .build()
        );
        
        // 5. 分块 & 向量化（交给下游异步处理）
        chunkAndEmbedAsync(entity.getId(), processed);
        
        long duration = System.currentTimeMillis() - startTime;
        
        return IngestionResult.builder()
            .documentId(entity.getId())
            .title((String) processed.getMetadata().get("title"))
            .qualityScore(qualityScore)
            .processingTime(duration)
            .warnings(processed.getErrors())
            .build();
    }
}
```

---

## 八、常见坑与最佳实践

### 8.1 PDF 解析的隐藏大坑

1. **CID 字体映射问题：** 某些 PDF 使用 CID 字体，PDFBox 提取出来是乱码。解决方案：嵌入字体文件或使用 Tika 兜底。
2. **竖排文字：** 中文竖排 PDF，提取结果顺序完全错乱。需要检测书写方向。
3. **加密 PDF：** 用户上传了带密码的 PDF。需要支持密码参数。
4. **超大型 PDF：** 500MB+ 的 PDF 直接 OOM。必须流式处理，按页加载。

### 8.2 Tesseract OCR 调优

```bash
# 安装中文语言包
sudo apt-get install tesseract-ocr-chi-sim

# PaddleOCR 作为备选（中文效果更好）
pip install paddlepaddle paddleocr
```

### 8.3 性能优化建议

| 优化项 | 方案 | 效果 |
|-------|------|------|
| PDF 解析 | 流式处理，按页加载 | 内存占用降低 80% |
| OCR | GPU 加速 + 预缓存 | 速度提升 10x |
| HTML 解析 | 先用 Jsoup 预处理，再转 Markdown | 减少无用内容 30% |
| 大文件 | 文件大小限制 + 分页处理 | 避免 OOM |
| 批量处理 | 线程池 + 背压控制 | 吞吐量提升 5x |

---

## 九、总结

数据预处理看似"脏活累活"，但它的质量直接决定你的 RAG 效果。我见过太多团队把 80% 的时间花在调 prompt 和换模型上，却忽略了最基础的数据清洗工作。

**记住这几条原则：**

1. **PDF 解析不要只依赖一种工具**，PDFBox + Tika + OCR 三级降级策略是标配
2. **表格是 PDF 的噩梦**，能提前用 Word 就不要扫描成 PDF
3. **元数据不是"锦上添花"**，它是后续权限过滤、搜索排序的基础
4. **建立质量评分机制**，低质量文档宁可不要，也别污染向量库
5. **责任链模式让管道可扩展**，新加一种格式只需新增一个 Processor

---

**往期回顾：**
- [企业级 RAG 系统架构设计：数据注入→检索→生成的全链路]()

---

**下篇预告：** 文档预处理完，怎么高效地同步到知识库？下一篇《知识库增量更新方案：实时同步企业文档变更的流水线设计》将彻底解决"改一个文档就要重建整个索引"的痛——CDC 监听数据库变更、文件系统 WatchService、增量向量化、旧向量清理策略，30 秒内新文档即可检索。关注我，别错过！

---

> 如果这篇文章帮你解决了一直以来"PDF 解析老大难"的问题，请点赞收藏关注三连。你的支持是我持续输出硬核干货的动力。
