# 用AI自动生成技术架构图：Draw.io不再手动拖拽的时代来了

> "服务器接收请求 → Nginx反向代理 → 网关层鉴权 → 业务服务 → Redis缓存 → MySQL读写分离 → 消息队列异步处理。"一句话描述完，AI在Draw.io里自动生成了一张标准的技术架构图。从2小时手动拖拽到30秒AI生成，这才是技术文档该有的效率。

## 一、产品核心体验

```
输入：自然语言描述系统架构
输出：Draw.io格式的架构图（可直接编辑）

示例输入：
"微服务架构：用户通过CDN访问前端，Nginx反向代理到API Gateway，
Gateway做鉴权和限流后路由到User Service、Order Service、Product Service三个微服务。
服务之间通过RabbitMQ异步通信。每个服务独立数据库。Redis做缓存。
ELK做日志收集，Prometheus+Grafana做监控。"

30秒后生成的架构图：
┌──────────────────────────────────────────────────────────┐
│                         [CDN]                             │
│                           ↓                               │
│                      [Nginx]                              │
│                           ↓                               │
│                   [API Gateway]                           │
│                (鉴权/限流/路由)                            │
│            ↙          ↓           ↘                       │
│   [User Service] [Order Service] [Product Service]        │
│        ↕              ↕              ↕                     │
│   [User DB]     [Order DB]     [Product DB]               │
│        ↓              ↓              ↓                     │
│   ┌─────────────────────────────────┐                     │
│   │         [RabbitMQ]              │                     │
│   └─────────────────────────────────┘                     │
│        ↕              ↕              ↕                     │
│   ┌─────────────────────────────────┐                     │
│   │       [Redis Cache]             │                     │
│   └─────────────────────────────────┘                     │
│        ↓              ↓              ↓                     │
│   ┌──────────────────────────────────────────────────┐    │
│   │       [ELK]  ←→  [Prometheus + Grafana]          │    │
│   │   日志收集           监控告警                       │    │
│   └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

## 二、技术实现

### 2.1 文字描述→Architecture Graph

```java
/**
 * 架构图解析引擎
 * 将自然语言描述转为结构化的架构图数据
 */
@Service
public class ArchitectureParser {
    
    @Autowired
    private ChatLanguageModel llm;
    
    /**
     * 解析架构描述
     */
    public ArchitectureGraph parse(String description) {
        
        String prompt = """
            # 角色
            你是一位资深系统架构师。请将以下架构描述转换为结构化的架构图数据。
            
            # 架构描述
            %s
            
            # 任务
            识别架构中的节点、层次、连接关系，输出结构化数据。
            
            # 节点类型规范
            - CLOUD: 云服务（AWS/阿里云/腾讯云等）
            - LB: 负载均衡（Nginx/F5/ALB等）
            - GATEWAY: API网关（Kong/Spring Cloud Gateway等）
            - SERVICE: 业务服务（Spring Boot/Node.js等）
            - DATABASE: 数据库（MySQL/PostgreSQL/MongoDB等）
            - CACHE: 缓存（Redis/Memcached等）
            - MQ: 消息队列（RabbitMQ/Kafka/RocketMQ等）
            - STORAGE: 存储（OSS/MinIO/CEPH等）
            - CDN: 内容分发
            - MONITOR: 监控告警（Prometheus/Grafana等）
            - LOG: 日志系统（ELK/Loki等）
            - FRONTEND: 前端（React/Vue/Angular等）
            - DNS: 域名解析
            - CONTAINER: 容器（Docker/K8s等）
            
            # 输出格式
            ```json
            {
              "layers": [
                {
                  "name": "层名称（如：接入层/网关层/服务层/数据层）",
                  "order": 1,
                  "color": "#EBF5FB",
                  "nodes": [
                    {
                      "id": "node-1",
                      "name": "节点显示名称",
                      "type": "节点类型",
                      "description": "节点描述",
                      "x": 300,
                      "y": 100,
                      "width": 160,
                      "height": 60,
                      "color": "#3498DB"
                    }
                  ]
                }
              ],
              "connections": [
                {
                  "id": "conn-1",
                  "from": "node-1",
                  "to": "node-2",
                  "label": "连接说明（可选）",
                  "type": "SOLID/DASHED",
                  "direction": "FORWARD/BI_DIRECTIONAL"
                }
              ],
              "groups": [
                {
                  "name": "组名称（如：微服务集群）",
                  "nodes": ["node-3", "node-4", "node-5"],
                  "color": "#F0F4F8",
                  "dashed": true
                }
              ]
            }
            ```
            
            只输出JSON。
            """.formatted(description);
        
        String response = llm.generate(prompt);
        String cleaned = response
            .replaceAll("```json\\s*", "")
            .replaceAll("```\\s*", "")
            .trim();
        
        return parseJSON(cleaned, ArchitectureGraph.class);
    }
}
```

### 2.2 自动布局算法

```java
/**
 * 自动布局引擎
 * 根据层级关系自动计算节点位置，无需手动指定坐标
 */
@Service
public class AutoLayoutEngine {
    
    /**
     * 为架构图的节点自动计算最优位置
     */
    public ArchitectureGraph autoLayout(ArchitectureGraph graph) {
        
        int canvasWidth = 1200;
        int marginLeft = 80;
        int marginTop = 40;
        int layerSpacing = 120;  // 层间距
        int nodeSpacing = 40;    // 同层节点间距
        int nodeWidth = 160;
        int nodeHeight = 60;
        
        // 如果有预定义的层次，按层次排列
        if (graph.getLayers() != null && !graph.getLayers().isEmpty()) {
            
            for (int layerIdx = 0; layerIdx < graph.getLayers().size(); layerIdx++) {
                Layer layer = graph.getLayers().get(layerIdx);
                List<Node> layerNodes = layer.getNodes();
                
                // 计算该层的总宽度
                int totalLayerWidth = layerNodes.size() * nodeWidth 
                    + (layerNodes.size() - 1) * nodeSpacing;
                int startX = marginLeft + (canvasWidth - marginLeft * 2 - totalLayerWidth) / 2;
                int y = marginTop + layerIdx * layerSpacing;
                
                for (int nodeIdx = 0; nodeIdx < layerNodes.size(); nodeIdx++) {
                    Node node = layerNodes.get(nodeIdx);
                    node.setX(startX + nodeIdx * (nodeWidth + nodeSpacing));
                    node.setY(y);
                    node.setWidth(nodeWidth);
                    node.setHeight(nodeHeight);
                }
            }
            
        } else {
            // 使用拓扑排序自动确定层次
            Map<String, Integer> nodeLevels = calculateNodeLevels(graph);
            Map<Integer, List<Node>> levelGroups = groupByLevel(graph.getNodes(), nodeLevels);
            
            for (Map.Entry<Integer, List<Node>> entry : levelGroups.entrySet()) {
                int level = entry.getKey();
                List<Node> nodesAtLevel = entry.getValue();
                
                int totalWidth = nodesAtLevel.size() * nodeWidth 
                    + (nodesAtLevel.size() - 1) * nodeSpacing;
                int startX = marginLeft + (canvasWidth - marginLeft * 2 - totalWidth) / 2;
                int y = marginTop + level * layerSpacing;
                
                for (int i = 0; i < nodesAtLevel.size(); i++) {
                    Node node = nodesAtLevel.get(i);
                    node.setX(startX + i * (nodeWidth + nodeSpacing));
                    node.setY(y);
                }
            }
        }
        
        return graph;
    }
    
    /**
     * 用拓扑排序计算节点层级
     */
    private Map<String, Integer> calculateNodeLevels(ArchitectureGraph graph) {
        Map<String, Integer> levels = new HashMap<>();
        Map<String, Integer> inDegree = new HashMap<>();
        Map<String, List<String>> adjList = new HashMap<>();
        
        // 构建邻接表
        for (Node node : graph.getNodes()) {
            inDegree.putIfAbsent(node.getId(), 0);
            adjList.putIfAbsent(node.getId(), new ArrayList<>());
        }
        
        for (Connection conn : graph.getConnections()) {
            adjList.get(conn.getFrom()).add(conn.getTo());
            inDegree.merge(conn.getTo(), 1, Integer::sum);
        }
        
        // BFS拓扑排序分配层级
        Queue<String> queue = new LinkedList<>();
        for (Map.Entry<String, Integer> entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) {
                queue.offer(entry.getKey());
                levels.put(entry.getKey(), 0);
            }
        }
        
        while (!queue.isEmpty()) {
            String current = queue.poll();
            int currentLevel = levels.get(current);
            
            for (String neighbor : adjList.get(current)) {
                int newLevel = currentLevel + 1;
                if (!levels.containsKey(neighbor) || levels.get(neighbor) < newLevel) {
                    levels.put(neighbor, newLevel);
                }
                inDegree.merge(neighbor, -1, Integer::sum);
                if (inDegree.get(neighbor) == 0) {
                    queue.offer(neighbor);
                }
            }
        }
        
        return levels;
    }
}
```

### 2.3 Draw.io XML生成

```java
/**
 * Draw.io格式生成器
 * 生成可被Draw.io直接打开的XML文件
 */
@Service
public class DrawioGenerator {
    
    /**
     * 生成Draw.io格式的XML
     */
    public String generate(ArchitectureGraph graph) {
        
        StringBuilder xml = new StringBuilder();
        xml.append("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
        xml.append("<mxfile host=\"app.diagrams.net\" modified=\"2025-01-01T00:00:00.000Z\" ");
        xml.append("agent=\"AI Architecture Generator\" version=\"24.0.0\">\n");
        xml.append("  <diagram name=\"").append(graph.getTitle()).append("\" ");
        xml.append("id=\"arch-diagram\">\n");
        xml.append("    <mxGraphModel dx=\"1434\" dy=\"790\" grid=\"1\" gridSize=\"10\" ");
        xml.append("guides=\"1\" tooltips=\"1\" connect=\"1\" arrows=\"1\" ");
        xml.append("fold=\"1\" page=\"1\" pageScale=\"1\" pageWidth=\"1400\" ");
        xml.append("pageHeight=\"1000\" math=\"0\" shadow=\"0\">\n");
        xml.append("      <root>\n");
        xml.append("        <mxCell id=\"0\"/>\n");
        xml.append("        <mxCell id=\"1\" parent=\"0\"/>\n");
        
        int cellId = 2;
        Map<String, String> nodeCellIds = new HashMap<>();
        
        // 绘制分组
        if (graph.getGroups() != null) {
            for (Group group : graph.getGroups()) {
                cellId = drawGroup(xml, group, graph, cellId);
            }
        }
        
        // 绘制连接线（先画线，再画节点，确保节点在上层）
        if (graph.getConnections() != null) {
            for (Connection conn : graph.getConnections()) {
                cellId = drawConnection(xml, conn, graph, cellId, nodeCellIds);
            }
        }
        
        // 绘制节点
        for (Layer layer : graph.getLayers()) {
            // 层标签
            cellId = drawLayerLabel(xml, layer, cellId);
            
            // 节点
            for (Node node : layer.getNodes()) {
                String cellIdStr = String.valueOf(cellId);
                nodeCellIds.put(node.getId(), cellIdStr);
                cellId = drawNode(xml, node, cellId);
            }
        }
        
        xml.append("      </root>\n");
        xml.append("    </mxGraphModel>\n");
        xml.append("  </diagram>\n");
        xml.append("</mxfile>");
        
        return xml.toString();
    }
    
    /**
     * 绘制单个节点
     */
    private int drawNode(StringBuilder xml, Node node, int cellId) {
        
        String color = node.getColor() != null ? node.getColor() : "#3498DB";
        String style = String.format(
            "rounded=1;whiteSpace=wrap;html=1;fillColor=%s;strokeColor=%s;fontColor=#FFFFFF;",
            color, darken(color, 0.2)
        );
        
        // 根据节点类型添加图标
        switch (node.getType()) {
            case "DATABASE":
                style += "shape=cylinder3;size=15;";
                break;
            case "CACHE":
                style += "shape=cylinder3;size=10;fillColor=#E74C3C;strokeColor=#C0392B;";
                break;
            case "MQ":
                style += "shape=parallelogram;perimeter=parallelogramPerimeter;fillColor=#F39C12;strokeColor=#E67E22;";
                break;
            case "LB":
                style += "shape=hexagon;perimeter=hexagonPerimeter2;size=0.2;fillColor=#9B59B6;strokeColor=#8E44AD;";
                break;
            case "GATEWAY":
                style += "shape=rhombus;perimeter=rhombusPerimeter;fillColor=#1ABC9C;strokeColor=#16A085;";
                break;
            case "FRONTEND":
                style += "shape=process;fillColor=#2ECC71;strokeColor=#27AE60;";
                break;
            default:
                break;
        }
        
        xml.append(String.format(
            "        <mxCell id=\"%d\" value=\"%s\" " +
            "style=\"%s\" vertex=\"1\" parent=\"1\">\n",
            cellId, escapeXml(node.getName()), style
        ));
        xml.append(String.format(
            "          <mxGeometry x=\"%d\" y=\"%d\" width=\"%d\" height=\"%d\" " +
            "as=\"geometry\"/>\n",
            (int) node.getX(), (int) node.getY(), 
            (int) node.getWidth(), (int) node.getHeight()
        ));
        xml.append("        </mxCell>\n");
        
        return cellId + 1;
    }
    
    /**
     * 绘制连接线
     */
    private int drawConnection(StringBuilder xml, Connection conn, 
                                ArchitectureGraph graph, int cellId,
                                Map<String, String> nodeCellIds) {
        
        String fromCellId = nodeCellIds.get(conn.getFrom());
        String toCellId = nodeCellIds.get(conn.getTo());
        
        if (fromCellId == null || toCellId == null) return cellId;
        
        String style = "edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;" +
            "jettySize=auto;html=1;exitX=0.5;exitY=1;exitDx=0;exitDy=0;" +
            "entryX=0.5;entryY=0;entryDx=0;entryDy=0;";
        
        if ("DASHED".equals(conn.getType())) {
            style += "dashed=1;";
        }
        
        if ("BI_DIRECTIONAL".equals(conn.getDirection())) {
            style += "startArrow=classic;endArrow=classic;";
        }
        
        xml.append(String.format(
            "        <mxCell id=\"%d\" value=\"%s\" " +
            "style=\"%s\" edge=\"1\" parent=\"1\" source=\"%s\" target=\"%s\">\n",
            cellId, 
            conn.getLabel() != null ? escapeXml(conn.getLabel()) : "",
            style, fromCellId, toCellId
        ));
        xml.append("          <mxGeometry relative=\"1\" as=\"geometry\"/>\n");
        xml.append("        </mxCell>\n");
        
        return cellId + 1;
    }
    
    /**
     * 绘制分组（虚线框）
     */
    private int drawGroup(StringBuilder xml, Group group, 
                           ArchitectureGraph graph, int cellId) {
        
        // 计算分组边界
        BoundingBox box = calculateBoundingBox(group.getNodes(), graph);
        
        String style = String.format(
            "rounded=1;whiteSpace=wrap;html=1;fillColor=%s;" +
            "strokeColor=%s;dashed=1;verticalAlign=top;align=left;spacingLeft=10;",
            group.getColor() != null ? group.getColor() : "#F0F4F8",
            "#CCCCCC"
        );
        
        xml.append(String.format(
            "        <mxCell id=\"%d\" value=\"%s\" " +
            "style=\"%s\" vertex=\"1\" parent=\"1\">\n",
            cellId, escapeXml(group.getName()), style
        ));
        xml.append(String.format(
            "          <mxGeometry x=\"%d\" y=\"%d\" width=\"%d\" height=\"%d\" " +
            "as=\"geometry\"/>\n",
            (int) box.x, (int) box.y, (int) box.width, (int) box.height
        ));
        xml.append("        </mxCell>\n");
        
        return cellId + 1;
    }
    
    private BoundingBox calculateBoundingBox(List<String> nodeIds, 
                                               ArchitectureGraph graph) {
        double minX = Double.MAX_VALUE, minY = Double.MAX_VALUE;
        double maxX = Double.MIN_VALUE, maxY = Double.MIN_VALUE;
        
        for (String nodeId : nodeIds) {
            Node node = findNode(graph, nodeId);
            if (node != null) {
                minX = Math.min(minX, node.getX());
                minY = Math.min(minY, node.getY());
                maxX = Math.max(maxX, node.getX() + node.getWidth());
                maxY = Math.max(maxY, node.getY() + node.getHeight());
            }
        }
        
        double padding = 20;
        return new BoundingBox(
            minX - padding,
            minY - padding - 30, // 留标签空间
            maxX - minX + padding * 2,
            maxY - minY + padding * 2 + 30
        );
    }
    
    record BoundingBox(double x, double y, double width, double height) {}
}
```

### 2.4 多格式导出

```java
/**
 * 多格式导出服务
 * 支持Draw.io XML、Mermaid、PlantUML等
 */
@Service
public class MultiFormatExporter {
    
    /**
     * 导出为Mermaid格式
     */
    public String exportToMermaid(ArchitectureGraph graph) {
        StringBuilder mmd = new StringBuilder();
        mmd.append("graph TD\n");
        
        // 样式定义
        for (Layer layer : graph.getLayers()) {
            for (Node node : layer.getNodes()) {
                String style = getMermaidStyle(node.getType());
                mmd.append(String.format("    classDef %s %s\n", 
                    node.getId(), style));
            }
        }
        
        // 子图定义
        for (Group group : graph.getGroups()) {
            mmd.append("    subgraph ").append(group.getName()).append("\n");
            for (String nodeId : group.getNodes()) {
                Node node = findNode(graph, nodeId);
                mmd.append("        ").append(nodeId)
                    .append("[").append(node.getName()).append("]\n");
            }
            mmd.append("    end\n");
        }
        
        // 连接
        for (Connection conn : graph.getConnections()) {
            String arrow = switch (conn.getDirection()) {
                case "BI_DIRECTIONAL" -> "<-->";
                case "FORWARD" -> "-->";
                default -> "-->";
            };
            
            String label = conn.getLabel() != null 
                ? "|" + conn.getLabel() + "|" : "";
            
            mmd.append(String.format("    %s %s%s %s\n", 
                conn.getFrom(), arrow, label, conn.getTo()));
        }
        
        return mmd.toString();
    }
    
    /**
     * 导出为PlantUML格式  
     */
    public String exportToPlantUML(ArchitectureGraph graph) {
        StringBuilder puml = new StringBuilder();
        puml.append("@startuml\n");
        puml.append("!theme plain\n\n");
        
        // 组件定义
        for (Node node : graph.getAllNodes()) {
            String component = switch (node.getType()) {
                case "DATABASE" -> "database";
                case "MQ" -> "queue";
                case "CACHE" -> "database";
                case "STORAGE" -> "storage";
                default -> "component";
            };
            
            puml.append(String.format("%s \"%s\" as %s #%s\n", 
                component, node.getName(), node.getId(), 
                node.getColor() != null ? node.getColor().replace("#", "") : "LightBlue"));
        }
        
        // 连接
        for (Connection conn : graph.getConnections()) {
            puml.append(String.format("%s --> %s", conn.getFrom(), conn.getTo()));
            if (conn.getLabel() != null) {
                puml.append(String.format(" : %s", conn.getLabel()));
            }
            puml.append("\n");
        }
        
        puml.append("@enduml");
        return puml.toString();
    }
}
```

## 四、商业应用

```
目标用户：
  1. 技术团队（架构评审、技术方案汇报）
  2. 售前团队（给客户出架构图，以前用Visio画2小时）
  3. 技术博主/讲师（文章配图、课程配图）
  4. 企业信息化部门（等保测评、ISO认证都要架构图）

定价模型：
  SaaS版：¥19.9/月（基础模板 + 10层复杂度）
  Pro版：¥49.9/月（自定义模板 + 品牌配色 + 50层复杂度）
  Team版：¥129/月（多人协作 + 团队模板库）

变现案例：
  某外包公司：售前方案配图时间从2小时降到10分钟 → 签约率提升15%
  某培训机构：课程架构图自动生成 → 课件制作效率提升5倍
```

## 五、写在最后

做架构图从来不是技术问题，是时间问题。没有一个架构师愿意花2小时画图，但他们需要图来沟通。

AI在这个场景的价值非常直接：**把"画图"的时间还给"思考架构"本身。**

---

*下期预告：**C03-AI PPT生成工具年入千万？我复刻了一个技术版，告诉你门道在哪**——AI PPT生成为什么能年入千万？技术门槛并不高，关键是场景精准。我用Java后端做了一个技术版的AI PPT生成器，还原了商业产品的核心功能。*

---

> **作者简介**：某大厂Java架构师转AI技术负责人，专注Java+AI工程化落地。关注我，每周一篇Java+AI硬核实战。
