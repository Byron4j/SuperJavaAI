# AI生成UI原型：Figma插件+GPT-4 Vision自动画界面，产品经理说需求AI直接出图

> "我想要一个电商App的订单详情页，要有订单状态、商品列表、物流信息和操作按钮。"30秒后，AI自动在Figma里生成了一套完整的UI原型。产品经理说需求，AI直接出图，设计师只需要微调。这篇文章拆解背后的技术原理和Java服务端实现。

## 一、产品体验

```
输入：自然语言描述
输出：Figma设计稿（自动创建Frame + 组件 + 样式）

示例输入：
"做一个移动端的登录页面，简约风格。
顶部是App Logo，中间是手机号和密码输入框，
下面有个登录按钮，底部有'忘记密码'和'注册账号'的链接。
背景是渐变色。"

输出（Figma自动生成）：
┌────────────────────┐
│                    │
│    渐变背景         │
│                    │
│    [App Logo]      │
│                    │
│  ┌────────────────┐│
│  │ 手机号          ││
│  └────────────────┘│
│  ┌────────────────┐│
│  │ 密码            ││
│  └────────────────┘│
│                    │
│  ┌────────────────┐│
│  │    登  录       ││
│  └────────────────┘│
│                    │
│ 忘记密码  注册账号  │
│                    │
└────────────────────┘
```

## 二、技术架构

```
┌─────────────────────────────────────────────────┐
│           AI UI生成系统                             │
├────────────┬───────────────┬────────────────────┤
│ 需求理解    │  UI结构生成    │ Figma API操作       │
│            │               │                    │
│ LLM将       │ AI设计布局    │ Figma REST API     │
│ 自然语言    │ 结构           │ 操作Figma文档       │
│ 转为结构化  │               │                    │
│ UI描述      │ 组件选择      │ Frame创建           │
│            │ (Button/      │                    │
│ UI元素      │  Input等)    │ 组件创建             │
│ 识别        │               │ (Rectangle/Text/    │
│            │ 样式方案      │  Button组件)         │
│ 布局推测    │ (颜色/字体/   │                    │
│            │  间距)        │ 样式设置             │
│            │               │ (填充/边框/阴影)      │
│            │               │                    │
│            │               │ 自动布局             │
│            │               │ (Auto Layout)       │
└────────────┴───────────────┴────────────────────┘
```

## 三、核心实现

### 3.1 自然语言→结构化UI描述

```java
/**
 * UI需求解析器
 * 将自然语言转为结构化的UI树描述
 */
@Service
public class UIRequirementParser {
    
    @Autowired
    private ChatLanguageModel llm;
    
    /**
     * 解析UI需求
     */
    public UIDesign parseToUIDesign(String requirement) {
        
        String prompt = """
            # 角色
            你是一位资深的UI/UX设计师。请将用户的需求描述转化为结构化的UI设计描述。
            
            # 用户需求
            %s
            
            # 设计原则
            1. 遵循移动端设计规范（iOS Human Interface / Material Design）
            2. 默认使用375pt宽度（iPhone标准）
            3. 组件间距遵循8pt网格系统
            4. 信息层级清晰：标题 > 内容 > 辅助信息
            5. 色彩方案合理：主色+辅色+中性色
            
            # 输出格式
            ```json
            {
              "pageName": "页面名称",
              "width": 375,
              "deviceType": "MOBILE/DESKTOP/TABLET",
              "backgroundColor": "#FFFFFF",
              "sections": [
                {
                  "type": "HEADER/BODY/FOOTER/DIVIDER",
                  "layout": "VERTICAL/HORIZONTAL/GRID/ABSOLUTE",
                  "padding": {"top": 16, "bottom": 16, "left": 16, "right": 16},
                  "gap": 12,
                  "backgroundColor": "#FFFFFF",
                  "components": [
                    {
                      "type": "TEXT/INPUT/BUTTON/IMAGE/ICON/LIST/CARD/AVATAR",
                      "label": "显示文本",
                      "style": {
                        "fontSize": 16,
                        "fontWeight": "REGULAR/BOLD/MEDIUM",
                        "color": "#333333",
                        "textAlign": "LEFT/CENTER/RIGHT",
                        "lineHeight": 1.5
                      },
                      "size": {
                        "width": 343,
                        "height": 48
                      },
                      "borderRadius": 8,
                      "backgroundColor": "#4A90D9",
                      "placeholder": "占位文本（如果是Input）",
                      "action": "交互说明（如果是Button）"
                    }
                  ]
                }
              ],
              "designSystem": {
                "primaryColor": "#4A90D9",
                "secondaryColor": "#50C878",
                "textPrimary": "#333333",
                "textSecondary": "#999999",
                "backgroundColor": "#F5F5F5",
                "cardBackground": "#FFFFFF",
                "borderColor": "#E0E0E0",
                "fontSize": {
                  "title": 20,
                  "subtitle": 16,
                  "body": 14,
                  "caption": 12
                },
                "spacing": {
                  "xs": 4,
                  "sm": 8,
                  "md": 12,
                  "lg": 16,
                  "xl": 24,
                  "xxl": 32
                },
                "borderRadius": {
                  "sm": 4,
                  "md": 8,
                  "lg": 12
                }
              }
            }
            ```
            
            只输出JSON，不要任何解释文字。
            """.formatted(requirement);
        
        String response = llm.generate(prompt);
        String cleaned = response
            .replaceAll("```json\\s*", "")
            .replaceAll("```\\s*", "")
            .trim();
        
        return parseJSON(cleaned, UIDesign.class);
    }
}
```

### 3.2 Figma API操作引擎

```java
/**
 * Figma API操作引擎
 * 通过Figma REST API自动创建设计稿
 */
@Service
@Slf4j
public class FigmaEngine {
    
    private final RestTemplate restTemplate;
    private final String figmaApiBase = "https://api.figma.com/v1";
    
    @Value("${figma.personal.token}")
    private String figmaToken;
    
    @Value("${figma.team.id}")
    private String teamId;
    
    /**
     * 完整流程：创建Figma文件 → 创建Frame → 添加组件
     */
    public FigmaResult generateUI(UIDesign design, String projectName) {
        
        try {
            // Step 1: 创建Figma文件
            String fileKey = createFigmaFile(projectName);
            
            // Step 2: 获取文件节点（需要一个初始Frame）
            String documentId = getDocumentId(fileKey);
            
            // Step 3: 根据UI设计结构创建Figma节点
            List<FigmaNode> nodes = convertToFigmaNodes(design);
            
            // Step 4: 批量创建节点（使用Figma的批量操作API）
            String result = createNodes(fileKey, documentId, nodes);
            
            return FigmaResult.builder()
                .success(true)
                .fileKey(fileKey)
                .fileUrl("https://www.figma.com/file/" + fileKey)
                .nodesCreated(nodes.size())
                .build();
                
        } catch (Exception e) {
            log.error("Figma生成失败", e);
            return FigmaResult.builder()
                .success(false)
                .error(e.getMessage())
                .build();
        }
    }
    
    /**
     * 将UI设计转为Figma节点树
     */
    private List<FigmaNode> convertToFigmaNodes(UIDesign design) {
        List<FigmaNode> nodes = new ArrayList<>();
        
        double currentY = 0;
        
        // 创建主Frame（画板）
        FigmaNode mainFrame = FigmaNode.builder()
            .type("FRAME")
            .name(design.getPageName())
            .x(0).y(0)
            .width(design.getWidth())
            .height(812) // iPhone默认高度
            .fills(List.of(createSolidFill(design.getBackgroundColor())))
            .constraints(Map.of("vertical", "TOP", "horizontal", "LEFT"))
            .children(new ArrayList<>())
            .build();
        
        // 处理每个Section
        for (UISection section : design.getSections()) {
            
            // Section容器
            FigmaNode sectionFrame = FigmaNode.builder()
                .type("FRAME")
                .name("Section-" + section.getType())
                .x(section.getPadding().getLeft())
                .y(currentY + section.getPadding().getTop())
                .width(design.getWidth() - section.getPadding().getLeft() 
                    - section.getPadding().getRight())
                .height(calculateSectionHeight(section))
                .layoutMode("VERTICAL")
                .itemSpacing(section.getGap())
                .paddingLeft(section.getPadding().getLeft())
                .paddingRight(section.getPadding().getRight())
                .paddingTop(section.getPadding().getTop())
                .paddingBottom(section.getPadding().getBottom())
                .fills(List.of(createSolidFill(section.getBackgroundColor())))
                .children(new ArrayList<>())
                .build();
            
            // 处理组件
            for (UIComponent component : section.getComponents()) {
                FigmaNode componentNode = createComponentNode(component, design);
                sectionFrame.getChildren().add(componentNode);
            }
            
            mainFrame.getChildren().add(sectionFrame);
            currentY += calculateSectionHeight(section);
        }
        
        nodes.add(mainFrame);
        return nodes;
    }
    
    /**
     * 创建单个组件节点
     */
    private FigmaNode createComponentNode(UIComponent component, UIDesign design) {
        
        return switch (component.getType().toUpperCase()) {
            case "TEXT" -> createTextNode(component, design);
            case "INPUT" -> createInputNode(component, design);
            case "BUTTON" -> createButtonNode(component, design);
            case "IMAGE" -> createImageNode(component, design);
            case "LIST", "CARD" -> createCardNode(component, design);
            case "AVATAR" -> createAvatarNode(component, design);
            case "ICON" -> createIconNode(component, design);
            case "DIVIDER" -> createDividerNode(component, design);
            default -> createTextNode(component, design);
        };
    }
    
    /**
     * 创建文本节点
     */
    private FigmaNode createTextNode(UIComponent component, UIDesign design) {
        return FigmaNode.builder()
            .type("TEXT")
            .name("Text-" + truncate(component.getLabel(), 20))
            .width(component.getSize().getWidth())
            .height(component.getSize().getHeight())
            .characters(component.getLabel())
            .fontSize(component.getStyle().getFontSize())
            .fontWeight(getFontWeight(component.getStyle().getFontWeight()))
            .textAlignHorizontal(
                getTextAlign(component.getStyle().getTextAlign()))
            .fills(List.of(createSolidFill(component.getStyle().getColor())))
            .lineHeightPx((int) (component.getStyle().getFontSize() 
                * component.getStyle().getLineHeight()))
            .build();
    }
    
    /**
     * 创建输入框节点
     */
    private FigmaNode createInputNode(UIComponent component, UIDesign design) {
        // 输入框 = 矩形背景 + 占位文本
        FigmaNode inputFrame = FigmaNode.builder()
            .type("FRAME")
            .name("Input-" + component.getPlaceholder())
            .width(component.getSize().getWidth())
            .height(component.getSize().getHeight())
            .fills(List.of(createSolidFill("#FFFFFF")))
            .strokes(List.of(createStroke(design.getDesignSystem().getBorderColor(), 1)))
            .cornerRadius(component.getBorderRadius())
            .layoutMode("HORIZONTAL")
            .paddingLeft(12)
            .paddingRight(12)
            .counterAxisAlignItems("CENTER")
            .children(new ArrayList<>())
            .build();
        
        // 占位文本
        FigmaNode placeholderText = FigmaNode.builder()
            .type("TEXT")
            .name("Placeholder")
            .characters(component.getPlaceholder())
            .fontSize(design.getDesignSystem().getFontSize().get("body"))
            .fills(List.of(createSolidFill(
                design.getDesignSystem().getTextSecondary())))
            .build();
        
        inputFrame.getChildren().add(placeholderText);
        return inputFrame;
    }
    
    /**
     * 创建按钮节点
     */
    private FigmaNode createButtonNode(UIComponent component, UIDesign design) {
        // 按钮 = 圆角矩形 + 文字
        FigmaNode buttonFrame = FigmaNode.builder()
            .type("FRAME")
            .name("Button-" + component.getLabel())
            .width(component.getSize().getWidth())
            .height(component.getSize().getHeight())
            .fills(List.of(createSolidFill(
                component.getBackgroundColor() != null 
                    ? component.getBackgroundColor() 
                    : design.getDesignSystem().getPrimaryColor())))
            .cornerRadius(component.getBorderRadius())
            .layoutMode("HORIZONTAL")
            .primaryAxisAlignItems("CENTER")
            .counterAxisAlignItems("CENTER")
            .children(new ArrayList<>())
            .build();
        
        // 按钮文字
        FigmaNode buttonText = FigmaNode.builder()
            .type("TEXT")
            .name("ButtonLabel")
            .characters(component.getLabel())
            .fontSize(component.getStyle().getFontSize())
            .fontWeight(600) // Semi-bold
            .fills(List.of(createSolidFill("#FFFFFF")))
            .textAlignHorizontal("CENTER")
            .build();
        
        buttonFrame.getChildren().add(buttonText);
        
        // 添加自动布局约束
        buttonFrame.setConstraints(Map.of(
            "horizontal", "SCALE",
            "vertical", "SCALE"
        ));
        
        return buttonFrame;
    }
    
    /**
     * 创建卡片节点
     */
    private FigmaNode createCardNode(UIComponent component, UIDesign design) {
        return FigmaNode.builder()
            .type("FRAME")
            .name("Card")
            .width(component.getSize().getWidth())
            .height(component.getSize().getHeight())
            .fills(List.of(createSolidFill(
                design.getDesignSystem().getCardBackground())))
            .cornerRadius(design.getDesignSystem().getBorderRadius().get("md"))
            .effects(List.of(createDropShadow()))
            .paddingLeft(16)
            .paddingRight(16)
            .paddingTop(12)
            .paddingBottom(12)
            .layoutMode("VERTICAL")
            .itemSpacing(8)
            .children(new ArrayList<>())
            .build();
    }
    
    /**
     * 调用Figma REST API创建节点
     */
    private String createNodes(String fileKey, String parentNodeId, 
                                List<FigmaNode> nodes) {
        
        StringBuilder json = new StringBuilder();
        json.append("{\"nodes\":[");
        
        for (int i = 0; i < nodes.size(); i++) {
            json.append(toFigmaJSON(nodes.get(i)));
            if (i < nodes.size() - 1) json.append(",");
        }
        json.append("]}");
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Figma-Token", figmaToken);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        String url = String.format(
            "%s/files/%s/nodes?parent_id=%s",
            figmaApiBase, fileKey, parentNodeId
        );
        
        ResponseEntity<String> response = restTemplate.postForEntity(
            url, new HttpEntity<>(json.toString(), headers), String.class
        );
        
        return response.getBody();
    }
    
    /**
     * 创建Figma文件
     */
    private String createFigmaFile(String name) {
        String url = figmaApiBase + "/files";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Figma-Token", figmaToken);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        Map<String, Object> body = Map.of(
            "name", name,
            "team_id", teamId
        );
        
        ResponseEntity<Map> response = restTemplate.postForEntity(
            url, new HttpEntity<>(body, headers), Map.class
        );
        
        return (String) response.getBody().get("key");
    }
    
    // 辅助方法
    private Map<String, Object> createSolidFill(String color) {
        return Map.of(
            "type", "SOLID",
            "color", hexToRgba(color)
        );
    }
    
    private Map<String, Object> createStroke(String color, int weight) {
        return Map.of(
            "type", "SOLID",
            "color", hexToRgba(color)
        );
    }
    
    private Map<String, Object> createDropShadow() {
        return Map.of(
            "type", "DROP_SHADOW",
            "color", Map.of("r", 0, "g", 0, "b", 0, "a", 0.1),
            "offset", Map.of("x", 0, "y", 2),
            "radius", 8,
            "visible", true
        );
    }
    
    private Map<String, Double> hexToRgba(String hex) {
        hex = hex.replace("#", "");
        return Map.of(
            "r", Integer.parseInt(hex.substring(0, 2), 16) / 255.0,
            "g", Integer.parseInt(hex.substring(2, 4), 16) / 255.0,
            "b", Integer.parseInt(hex.substring(4, 6), 16) / 255.0,
            "a", 1.0
        );
    }
}

/**
 * Figma节点数据模型
 */
@Data
@Builder
public class FigmaNode {
    private String type;       // FRAME/TEXT/RECTANGLE等
    private String name;
    private double x, y;
    private double width, height;
    private String characters; // 文本内容
    private int fontSize;
    private int fontWeight;
    private String textAlignHorizontal;
    private double lineHeightPx;
    private List<Map<String, Object>> fills;
    private List<Map<String, Object>> strokes;
    private List<Map<String, Object>> effects;
    private double cornerRadius;
    private String layoutMode; // VERTICAL/HORIZONTAL/NONE
    private double itemSpacing;
    private double paddingLeft, paddingRight, paddingTop, paddingBottom;
    private String primaryAxisAlignItems; // CENTER/MIN/MAX
    private String counterAxisAlignItems; // CENTER/MIN/MAX
    private Map<String, String> constraints;
    private List<FigmaNode> children;
}
```

## 四、Figma Plugin 集成（前端部分）

```javascript
// Figma插件代码 (code.ts)
// 接收来自Java后端的UI描述，在Figma中创建节点

figma.showUI(__html__, { width: 400, height: 600 });

figma.ui.onmessage = async (msg) => {
  
  if (msg.type === 'generate-ui') {
    const design = msg.design;
    
    // 创建设计系统颜色样式
    setupDesignSystem(design.designSystem);
    
    // 创建主Frame
    const mainFrame = figma.createFrame();
    mainFrame.name = design.pageName;
    mainFrame.resize(design.width, 812);
    mainFrame.fills = [{ type: 'SOLID', color: hexToRgb(design.backgroundColor) }];
    
    let yOffset = 0;
    
    // 遍历Section创建UI
    for (const section of design.sections) {
      const sectionFrame = createSection(section, design);
      sectionFrame.y = yOffset;
      mainFrame.appendChild(sectionFrame);
      yOffset += sectionFrame.height;
    }
    
    // 选中并缩放到适合
    figma.currentPage.appendChild(mainFrame);
    figma.viewport.scrollAndZoomIntoView([mainFrame]);
    figma.closePlugin();
  }
};

function createComponent(component, design) {
  switch (component.type) {
    case 'TEXT':
      return createText(component);
    case 'INPUT':
      return createInput(component, design);
    case 'BUTTON':
      return createButton(component, design);
    case 'CARD':
      return createCard(component, design);
    case 'IMAGE':
      return createImagePlaceholder(component);
    case 'AVATAR':
      return createAvatar(component);
    default:
      return createText(component);
  }
}

function createButton(component, design) {
  const frame = figma.createFrame();
  frame.name = 'Button';
  frame.resize(component.size.width, component.size.height);
  frame.cornerRadius = component.borderRadius;
  frame.fills = [{
    type: 'SOLID',
    color: hexToRgb(component.backgroundColor || design.designSystem.primaryColor)
  }];
  frame.primaryAxisAlignItems = 'CENTER';
  frame.counterAxisAlignItems = 'CENTER';
  
  const text = figma.createText();
  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
  text.fontName = { family: "Inter", style: "Semi Bold" };
  text.characters = component.label;
  text.fontSize = component.style.fontSize;
  text.fills = [{ type: 'SOLID', color: hexToRgb('#FFFFFF') }];
  
  frame.appendChild(text);
  frame.layoutMode = 'HORIZONTAL';
  return frame;
}
```

## 五、商业应用场景

```
场景1：产品需求评审
  产品经理想法 → AI出UI原型 → 20分钟完成需求可视化 → 研发评估更准确

场景2：甲方演示
  售前团队现场输入需求 → AI实时出图 → 甲方"就是这个感觉！"

场景3：设计稿批量生成
  后台管理系统的50个页面 → AI一键生成 → 设计师只做关键页面

场景4：UI自动化测试
  设计稿 → 自动生成React组件 → 开发直接用（下篇C05会讲到）
```

## 六、写在最后

AI生成UI原型最大的意义不是替代设计师，而是**缩短"想法→视觉"的距离**。以前产品经理要画线框图→出PRD→设计出图→评审→修改，循环2-3周。现在产品经理说一句话，30秒出图，发现问题当场改。

**这不是效率工具，这是协作方式的革命。**

---

*下期预告：**C02-用AI自动生成技术架构图：Draw.io不再手动拖拽的时代来了**——传统架构图手动拖拽2小时，用AI Text-to-Diagram 30秒生成。我会分享如何用AI将文字描述自动转换为Draw.io格式的架构图，以及Mermaid代码的自动生成。*

---

> **作者简介**：某大厂Java架构师转AI技术负责人，专注Java+AI工程化落地。关注我，每周一篇Java+AI硬核实战。
