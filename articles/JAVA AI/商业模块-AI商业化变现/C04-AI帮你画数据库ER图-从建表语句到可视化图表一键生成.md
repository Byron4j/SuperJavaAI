# AI帮你画数据库ER图：从建表语句到可视化图表一键生成

> 数据库设计完成后，最烦的就是画ER图。100张表手动拖3天，改一个字段重画一遍。我用AI做了一个工具：粘贴DDL建表语句，自动识别实体关系，生成专业的ER图。支持导出PNG、SVG、Draw.io格式。这篇文章给你完整的技术实现。

## 一、效果展示

```
输入：MySQL DDL建表语句

输出：自动生成的ER图
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   users     │       │   orders    │       │  products   │
├─────────────┤       ├─────────────┤       ├─────────────┤
│🔑 id (PK)   │───1:N─│🔑 id (PK)   │       │🔑 id (PK)   │
│  username   │       │🔗 user_id  │───N:1─│  name       │
│  email      │       │  total_amt  │       │  price      │
│  phone      │       │  status     │       │  stock      │
│  created_at │       │  created_at │       │🔗 category_id│
└─────────────┘       └──────┬──────┘       └──────┬──────┘
                             │                     │
                             │ 1:N                 │ N:1
                             │                     │
                      ┌──────┴──────┐       ┌──────┴──────┐
                      │order_items │       │  categories │
                      ├─────────────┤       ├─────────────┤
                      │🔑 id (PK)   │       │🔑 id (PK)   │
                      │🔗 order_id  │       │  name       │
                      │🔗 product_id│       │  parent_id  │
                      │  quantity   │       │  sort_order │
                      │  price      │       └─────────────┘
                      └─────────────┘
```

## 二、技术架构

```
┌──────────────────────────────────────────────┐
│           AI ER图生成系统                       │
├──────────┬────────────┬──────────┬───────────┤
│ DDL解析   │ 关系推断    │ 布局计算  │ 图像渲染   │
│          │            │          │           │
│ SQL解析器│ FK关系      │ Sugiyama │ SVG渲染   │
│ 提取表名 │ 命名推断    │ 算法     │           │
│ 字段、类型│ (user_id→  │          │ PNG导出   │
│ 约束     │  关联user) │ 分簇优化  │ (Batík)  │
│          │            │          │           │
│ AI辅助   │ 语义推断    │ 手动调整  │ Draw.io   │
│ 不完整DDL│ (order_items│ 后自动   │ 导出      │
│ 补全     │ →order)    │ 重新计算  │           │
└──────────┴────────────┴──────────┴───────────┘
```

## 三、核心实现

### 3.1 DDL解析器

```java
/**
 * DDL解析器
 * 从SQL DDL中提取表结构信息
 */
@Service
public class DDLParser {
    
    /**
     * 解析建表语句，提取表结构
     */
    public List<TableInfo> parse(String ddl) {
        
        List<TableInfo> tables = new ArrayList<>();
        
        // 正则匹配 CREATE TABLE 语句
        Pattern tablePattern = Pattern.compile(
            "CREATE\\s+TABLE\\s+(?:IF\\s+NOT\\s+EXISTS\\s+)?`?(\\w+)`?\\s*\\(([\\s\\S]*?)\\)\\s*[^)]*?(?:;|$)",
            Pattern.CASE_INSENSITIVE
        );
        
        Matcher tableMatcher = tablePattern.matcher(ddl);
        
        while (tableMatcher.find()) {
            String tableName = tableMatcher.group(1);
            String columnsBlock = tableMatcher.group(2);
            
            TableInfo table = TableInfo.builder()
                .name(tableName)
                .columns(new ArrayList<>())
                .indexes(new ArrayList<>())
                .build();
            
            // 解析列定义
            parseColumns(columnsBlock, table);
            
            // 解析索引
            parseIndexes(columnsBlock, table);
            
            // 解析表注释
            parseTableComment(ddl, table);
            
            tables.add(table);
        }
        
        return tables;
    }
    
    /**
     * 解析列定义
     */
    private void parseColumns(String columnsBlock, TableInfo table) {
        
        // 匹配列定义：column_name TYPE [约束] [COMMENT 'xxx']
        Pattern columnPattern = Pattern.compile(
            "`?(\\w+)`?\\s+(\\w+(?:\\([^)]*\\))?)\\s*(.*?)(?=,|$)",
            Pattern.CASE_INSENSITIVE
        );
        
        Matcher columnMatcher = columnPattern.matcher(columnsBlock);
        
        while (columnMatcher.find()) {
            String columnName = columnMatcher.group(1);
            String columnType = columnMatcher.group(2);
            String constraints = columnMatcher.group(3);
            
            // 跳过PRIMARY KEY, INDEX等非列定义
            if (isKeyword(columnName)) continue;
            
            ColumnInfo column = ColumnInfo.builder()
                .name(columnName)
                .type(parseType(columnType))
                .length(parseLength(columnType))
                .nullable(!constraints.toUpperCase().contains("NOT NULL"))
                .primaryKey(constraints.toUpperCase().contains("PRIMARY KEY"))
                .autoIncrement(constraints.toUpperCase().contains("AUTO_INCREMENT"))
                .unique(constraints.toUpperCase().contains("UNIQUE"))
                .defaultValue(extractDefaultValue(constraints))
                .comment(extractComment(constraints))
                .build();
            
            // 判断是否是外键（通过命名约定）
            if (columnName.endsWith("_id") && column.isPrimaryKey() == false) {
                String referencedTable = columnName.substring(0, columnName.length() - 3)
                    .replaceAll("_", "");
                column.setPossibleForeignKey(true);
                column.setReferencedTable(referencedTable);
            }
            
            // 判断是否是枚举字段
            if (columnType.toUpperCase().startsWith("ENUM")) {
                column.setEnumValues(parseEnumValues(columnType));
            }
            
            table.getColumns().add(column);
        }
    }
    
    /**
     * 解析外键约束
     */
    private void parseForeignKeyConstraints(String columnsBlock, 
                                              List<TableInfo> allTables) {
        
        // 匹配 FOREIGN KEY (col) REFERENCES table(col)
        Pattern fkPattern = Pattern.compile(
            "FOREIGN\\s+KEY\\s*\\(`?(\\w+)`?\\)\\s*REFERENCES\\s+`?(\\w+)`?\\s*\\(`?(\\w+)`?\\)",
            Pattern.CASE_INSENSITIVE
        );
        
        Matcher fkMatcher = fkPattern.matcher(columnsBlock);
        
        while (fkMatcher.find()) {
            String fkColumn = fkMatcher.group(1);
            String referencedTable = fkMatcher.group(2);
            String referencedColumn = fkMatcher.group(3);
            
            // 建立关系
            // ...
        }
    }
}
```

### 3.2 关系推断引擎

```java
/**
 * 关系推断引擎
 * 通过AI + 规则自动推断表间关系
 */
@Service
public class RelationshipInferenceEngine {
    
    @Autowired
    private ChatLanguageModel llm;
    
    /**
     * 推断表间关系
     */
    public List<TableRelationship> inferRelationships(List<TableInfo> tables) {
        
        List<TableRelationship> relationships = new ArrayList<>();
        
        // Step 1: 基于命名约定的推断（确定性规则）
        relationships.addAll(inferByNameConvention(tables));
        
        // Step 2: 基于语义的推断（AI辅助）
        relationships.addAll(inferBySemantics(tables, relationships));
        
        // Step 3: 去重和消歧
        return deduplicateRelationships(relationships);
    }
    
    /**
     * 基于命名约定的关系推断
     */
    private List<TableRelationship> inferByNameConvention(List<TableInfo> tables) {
        
        List<TableRelationship> relationships = new ArrayList<>();
        
        for (TableInfo table : tables) {
            for (ColumnInfo column : table.getColumns()) {
                
                // 规则1: xxx_id → 关联 xxx 表
                if (column.getName().endsWith("_id") && !column.isPrimaryKey()) {
                    String possibleTableName = column.getName()
                        .replace("_id", "");
                    
                    // 尝试找到对应的表
                    for (TableInfo targetTable : tables) {
                        if (targetTable.getName().equalsIgnoreCase(possibleTableName)
                            || toPlural(targetTable.getName())
                                .equalsIgnoreCase(toPlural(possibleTableName))) {
                            
                            relationships.add(TableRelationship.builder()
                                .sourceTable(table.getName())
                                .sourceColumn(column.getName())
                                .targetTable(targetTable.getName())
                                .targetColumn("id")
                                .type("N:1")
                                .inferredBy("NAMING_CONVENTION")
                                .build());
                        }
                    }
                }
                
                // 规则2: 中间表识别（两个外键 + 无其他业务字段 = 多对多中间表）
                // order_items: order_id, product_id → orders <-> products (M:N)
            }
        }
        
        // 识别多对多中间表
        relationships.addAll(inferManyToMany(tables));
        
        return relationships;
    }
    
    /**
     * 识别多对多中间表
     */
    private List<TableRelationship> inferManyToMany(List<TableInfo> tables) {
        
        List<TableRelationship> relationships = new ArrayList<>();
        
        for (TableInfo table : tables) {
            List<ColumnInfo> fks = table.getColumns().stream()
                .filter(c -> c.getName().endsWith("_id") && !c.isPrimaryKey())
                .collect(Collectors.toList());
            
            // 有恰好2个外键，且其他字段少于3个 → 很可能是中间表
            if (fks.size() == 2) {
                long otherFields = table.getColumns().stream()
                    .filter(c -> !c.isPrimaryKey())
                    .filter(c -> !c.getName().endsWith("_id"))
                    .count();
                
                if (otherFields <= 3) {
                    // 识别为多对多中间表
                    for (ColumnInfo fk : fks) {
                        String refTable = findReferencedTable(fk, tables);
                        if (refTable != null) {
                            relationships.add(TableRelationship.builder()
                                .sourceTable(table.getName())
                                .sourceColumn(fk.getName())
                                .targetTable(refTable)
                                .targetColumn("id")
                                .type("N:1")
                                .isIntermediate(true)
                                .inferredBy("PATTERN_MANY_TO_MANY")
                                .build());
                        }
                    }
                }
            }
        }
        
        return relationships;
    }
    
    /**
     * AI语义推断：处理命名不明显的关系
     */
    private List<TableRelationship> inferBySemantics(
            List<TableInfo> tables, 
            List<TableRelationship> existingRelationships) {
        
        // 只对尚未建立关系的表对进行AI推断
        Set<String> existingPairs = existingRelationships.stream()
            .map(r -> r.getSourceTable() + "->" + r.getTargetTable())
            .collect(Collectors.toSet());
        
        List<TableRelationship> inferred = new ArrayList<>();
        
        for (TableInfo source : tables) {
            for (TableInfo target : tables) {
                if (source == target) continue;
                
                String pair = source.getName() + "->" + target.getName();
                if (existingPairs.contains(pair)) continue;
                
                // AI判断表间是否有关系
                String prompt = """
                    分析以下两个数据库表，判断它们之间是否存在业务关系。
                    
                    表A: %s
                    字段: %s
                    
                    表B: %s  
                    字段: %s
                    
                    如果存在关系，输出：
                    {
                      "hasRelationship": true,
                      "type": "1:1/1:N/N:1/M:N",
                      "reason": "关系推断理由"
                    }
                    
                    如果不存在关系，输出：
                    {
                      "hasRelationship": false
                    }
                    """.formatted(
                        source.getName(), formatColumns(source.getColumns()),
                        target.getName(), formatColumns(target.getColumns())
                    );
                
                String response = llm.generate(prompt);
                Map result = parseJSON(response, Map.class);
                
                if (Boolean.TRUE.equals(result.get("hasRelationship"))) {
                    inferred.add(TableRelationship.builder()
                        .sourceTable(source.getName())
                        .targetTable(target.getName())
                        .type((String) result.get("type"))
                        .inferredBy("AI_SEMANTIC")
                        .reason((String) result.get("reason"))
                        .build());
                }
            }
        }
        
        return inferred;
    }
}
```

### 3.3 自动布局与ER图渲染

```java
/**
 * ER图布局与SVG渲染
 */
@Service
public class ERDiagramRenderer {
    
    private static final int TABLE_WIDTH = 220;
    private static final int ROW_HEIGHT = 28;
    private static final int HEADER_HEIGHT = 36;
    private static final int TABLE_SPACING_X = 60;
    private static final int TABLE_SPACING_Y = 80;
    
    /**
     * 渲染ER图为SVG
     */
    public String renderToSVG(List<TableInfo> tables, 
                               List<TableRelationship> relationships,
                               DiagramLayout layout) {
        
        StringBuilder svg = new StringBuilder();
        
        // 计算画布大小
        int canvasWidth = 1200;
        int canvasHeight = 1000;
        
        svg.append(String.format(
            "<svg xmlns=\"http://www.w3.org/2000/svg\" " +
            "width=\"%d\" height=\"%d\" " +
            "viewBox=\"0 0 %d %d\">\n",
            canvasWidth, canvasHeight, canvasWidth, canvasHeight
        ));
        
        // 样式定义
        svg.append("<defs>\n");
        svg.append("  <filter id=\"shadow\">\n");
        svg.append("    <feDropShadow dx=\"2\" dy=\"2\" stdDeviation=\"3\" " +
            "flood-opacity=\"0.1\"/>\n");
        svg.append("  </filter>\n");
        svg.append("</defs>\n");
        
        // 背景
        svg.append("<rect width=\"100%\" height=\"100%\" " +
            "fill=\"#F8F9FA\"/>\n");
        
        // 绘制连接线
        for (TableRelationship rel : relationships) {
            drawRelationshipLine(svg, rel, layout.getNodePositions());
        }
        
        // 绘制表
        for (TableInfo table : tables) {
            NodePosition pos = layout.getNodePosition(table.getName());
            if (pos == null) continue;
            
            drawTableSVG(svg, table, pos);
        }
        
        svg.append("</svg>");
        return svg.toString();
    }
    
    /**
     * 绘制表格SVG
     */
    private void drawTableSVG(StringBuilder svg, TableInfo table, NodePosition pos) {
        
        int tableHeight = HEADER_HEIGHT + table.getColumns().size() * ROW_HEIGHT;
        
        // 表格阴影和背景
        svg.append(String.format(
            "<g transform=\"translate(%.0f, %.0f)\" filter=\"url(#shadow)\">\n",
            pos.getX(), pos.getY()
        ));
        
        // 表头
        svg.append(String.format(
            "<rect x=\"0\" y=\"0\" width=\"%d\" height=\"%d\" " +
            "rx=\"6\" ry=\"6\" fill=\"#3498DB\"/>\n",
            TABLE_WIDTH, HEADER_HEIGHT
        ));
        
        // 表头文字
        svg.append(String.format(
            "<text x=\"%d\" y=\"%d\" fill=\"white\" " +
            "font-family=\"Arial\" font-size=\"14\" font-weight=\"bold\" " +
            "text-anchor=\"middle\" dominant-baseline=\"central\">%s</text>\n",
            TABLE_WIDTH / 2, HEADER_HEIGHT / 2,
            escapeXml(table.getName())
        ));
        
        // 表格主体
        svg.append(String.format(
            "<rect x=\"0\" y=\"%d\" width=\"%d\" height=\"%d\" " +
            "fill=\"white\" stroke=\"#E0E0E0\" stroke-width=\"1\"/>\n",
            HEADER_HEIGHT, TABLE_WIDTH, tableHeight - HEADER_HEIGHT
        ));
        
        // 列
        for (int i = 0; i < table.getColumns().size(); i++) {
            ColumnInfo col = table.getColumns().get(i);
            int y = HEADER_HEIGHT + i * ROW_HEIGHT;
            
            // 行分隔线
            if (i > 0) {
                svg.append(String.format(
                    "<line x1=\"0\" y1=\"%d\" x2=\"%d\" y2=\"%d\" " +
                    "stroke=\"#F0F0F0\" stroke-width=\"1\"/>\n",
                    y, TABLE_WIDTH, y
                ));
            }
            
            // 主键标识
            String prefix = col.isPrimaryKey() ? "🔑 " : 
                           col.isPossibleForeignKey() ? "🔗 " : "   ";
            
            // 字段名
            svg.append(String.format(
                "<text x=\"12\" y=\"%d\" fill=\"%s\" " +
                "font-family=\"Consolas,monospace\" font-size=\"12\" " +
                "dominant-baseline=\"central\">%s%s</text>\n",
                y + ROW_HEIGHT / 2,
                col.isPrimaryKey() ? "#E74C3C" : 
                col.isPossibleForeignKey() ? "#2ECC71" : "#333333",
                prefix, col.getName()
            ));
            
            // 字段类型
            String typeStr = "(" + col.getType() + 
                (col.getLength() > 0 ? "(" + col.getLength() + ")" : "") + ")";
            
            svg.append(String.format(
                "<text x=\"%d\" y=\"%d\" fill=\"#999999\" " +
                "font-family=\"Consolas,monospace\" font-size=\"10\" " +
                "text-anchor=\"end\" dominant-baseline=\"central\">%s</text>\n",
                TABLE_WIDTH - 12, y + ROW_HEIGHT / 2, typeStr
            ));
        }
        
        svg.append("</g>\n");
    }
    
    /**
     * 绘制关系连接线
     */
    private void drawRelationshipLine(StringBuilder svg, 
                                        TableRelationship rel,
                                        Map<String, NodePosition> positions) {
        
        NodePosition from = positions.get(rel.getSourceTable());
        NodePosition to = positions.get(rel.getTargetTable());
        
        if (from == null || to == null) return;
        
        // 计算连接点
        EdgePoint edge = calculateEdgePoints(from, to);
        
        // 绘制贝塞尔曲线
        svg.append(String.format(
            "<path d=\"M %.0f %.0f C %.0f %.0f, %.0f %.0f, %.0f %.0f\" " +
            "stroke=\"#BDC3C7\" stroke-width=\"1.5\" fill=\"none\" " +
            "marker-end=\"url(#arrow-%s)\"/>\n",
            edge.getStartX(), edge.getStartY(),
            edge.getCtrl1X(), edge.getCtrl1Y(),
            edge.getCtrl2X(), edge.getCtrl2Y(),
            edge.getEndX(), edge.getEndY(),
            rel.getType()
        ));
        
        // 关系标注
        double labelX = (edge.getStartX() + edge.getEndX()) / 2;
        double labelY = (edge.getStartY() + edge.getEndY()) / 2 - 10;
        
        String label = switch (rel.getType()) {
            case "1:1" -> "1:1";
            case "1:N" -> "1:N";
            case "N:1" -> "N:1";
            case "M:N" -> "M:N";
            default -> "";
        };
        
        svg.append(String.format(
            "<text x=\"%.0f\" y=\"%.0f\" fill=\"#7F8C8D\" " +
            "font-family=\"Arial\" font-size=\"11\" " +
            "text-anchor=\"middle\">%s</text>\n",
            labelX, labelY, label
        ));
    }
}
```

### 3.4 Draw.io格式导出

```java
/**
 * Draw.io XML格式导出
 * 直接把ER图导出为Draw.io可编辑的文件
 */
@Service
public class ERDiagramDrawioExporter {
    
    public String exportToDrawio(List<TableInfo> tables, 
                                  List<TableRelationship> relationships,
                                  DiagramLayout layout) {
        
        StringBuilder xml = new StringBuilder();
        xml.append("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
        xml.append("<mxfile host=\"app.diagrams.net\">\n");
        xml.append("  <diagram name=\"ER Diagram\">\n");
        xml.append("    <mxGraphModel>\n");
        xml.append("      <root>\n");
        xml.append("        <mxCell id=\"0\"/>\n");
        xml.append("        <mxCell id=\"1\" parent=\"0\"/>\n");
        
        int cellId = 2;
        Map<String, String> tableCellIds = new HashMap<>();
        
        // 绘制关系连线
        for (TableRelationship rel : relationships) {
            String fromId = tableCellIds.get(rel.getSourceTable());
            String toId = tableCellIds.get(rel.getTargetTable());
            
            if (fromId != null && toId != null) {
                xml.append(String.format(
                    "        <mxCell id=\"%d\" style=\"edgeStyle=orthogonalEdgeStyle;" +
                    "rounded=0;html=1;endArrow=classic;endFill=1;\" " +
                    "edge=\"1\" parent=\"1\" source=\"%s\" target=\"%s\">\n",
                    cellId++, fromId, toId
                ));
                xml.append("        </mxCell>\n");
            }
        }
        
        // 绘制表节点（每个表是一个带SQL代码的矩形）
        for (TableInfo table : tables) {
            NodePosition pos = layout.getNodePosition(table.getName());
            if (pos == null) continue;
            
            String cellIdStr = String.valueOf(cellId);
            tableCellIds.put(table.getName(), cellIdStr);
            
            String tableContent = formatTableForDrawio(table);
            
            xml.append(String.format(
                "        <mxCell id=\"%d\" value=\"%s\" " +
                "style=\"shape=table;startSize=30;container=1;collapsible=0;" +
                "childLayout=tableLayout;fontStyle=1;align=center;" +
                "fillColor=#3498DB;fontColor=#FFFFFF;strokeColor=#2980B9;\" " +
                "vertex=\"1\" parent=\"1\">\n",
                cellId++, escapeXml(tableContent)
            ));
            xml.append(String.format(
                "          <mxGeometry x=\"%.0f\" y=\"%.0f\" " +
                "width=\"%d\" height=\"%d\" as=\"geometry\"/>\n",
                pos.getX(), pos.getY(), TABLE_WIDTH,
                HEADER_HEIGHT + table.getColumns().size() * ROW_HEIGHT
            ));
            xml.append("        </mxCell>\n");
        }
        
        xml.append("      </root>\n");
        xml.append("    </mxGraphModel>\n");
        xml.append("  </diagram>\n");
        xml.append("</mxfile>");
        
        return xml.toString();
    }
}
```

## 四、商业应用

```
目标场景：
  1. 数据库设计评审 → 粘贴DDL，自动生成可视化ER图
  2. 遗留系统文档化 → 从生产数据库反向生成ER图
  3. 技术方案PPT → 导出高清PNG用于方案汇报
  4. API文档配图 → 自动生成数据模型图

产品形态：
  - Web在线版（粘贴DDL → 生成 → 导出）
  - IDE插件（IntelliJ IDEA中选中.sql文件 → 右键生成ER图）
  - CLI工具（CI/CD中自动生成 → 写入项目docs目录）
```

## 五、写在最后

数据库ER图生成是一个典型的"技术不难但没人做得好"的场景。AI在这里的作用不是生成图（渲染是确定性算法），而是**理解表间关系**。命名约定的规则只能覆盖80%的情况，剩下20%的隐式关系需要AI的语义理解来补全。

**AI+传统工具 = 最好的产品。不是AI替代一切，而是AI填补传统工具做不到的缺口。**

---

*下期预告：**C05-设计稿→前端代码：Screenshot-to-Code的Java服务端实现**——有一个很火的AI开源项目叫screenshot-to-code，能把UI截图转成前端代码。我用Java做了服务端实现，把设计稿直接变成React组件代码。这篇文章揭示背后的技术原理。*

---

> **作者简介**：某大厂Java架构师转AI技术负责人，专注Java+AI工程化落地。关注我，每周一篇Java+AI硬核实战。
