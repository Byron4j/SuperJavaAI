# 餐饮菜单AI优化：定价+排列+文案一体化工具

> 餐厅菜单不是随便写的—定价心理学、视觉热区、文案设计，每个细节都在影响客单价。我们用AI帮餐厅优化菜单，客单价平均提升15%-28%。

---

## 一、行业痛点：菜单设计的"反常识"

### 一个餐厅老板的困惑

去年底，杭州一家日料店的老板找到我，说他的店菜品质量没得说，但客单价就是上不去。人均消费一直在¥120左右徘徊，而隔壁同类日料店能做到¥170。

我让他把菜单发给我看，问题一目了然：

```
该日料店菜单问题诊断：

问题1: 价格锚点缺失
  → 最贵的菜是¥188的刺身拼盘，第二贵直接掉到¥98
  → 缺少"锚定效应"：没有超高价的菜来衬托其他菜"便宜"

问题2: 菜品顺序混乱
  → 第一页第一道菜是¥28的毛豆
  → 菜单视觉热区（右上角）放的是饮品列表
  → 高价菜藏在第4页

问题3: 菜品描述白开水
  → "三文鱼刺身 ¥58"
  → 没有任何场景描述、产地信息、口感形容词
  → 顾客无法感知价值

问题4: 套餐设计不合理
  → 单人套餐¥88、双人套餐¥298 → 两个人各点单人套餐只要¥176
  → 套餐价格倒挂，聪明的顾客不会点双人套餐

问题5: 定价尾数混乱
  → 有的¥58，有的¥60，有的¥65
  → 没有使用心理学定价（¥58比¥60显得便宜，实际只差2元）
```

**数据佐证**：餐饮行业研究显示，专业的菜单优化可以让客单价提升15%-30%。对于一家月流水30万的中型餐厅，这意味着每年多赚54万-108万。

---

## 二、菜单优化四要素

### 菜单设计背后的科学

```
┌─────────────────────────────────────────────────────┐
│              餐厅菜单优化四大维度                       │
├───────────────┬──────────────┬──────────────────────┤
│   定价策略     │   排列布局    │    文案设计           │
│   (35%)       │   (25%)      │    (25%)             │
├───────────────┼──────────────┼──────────────────────┤
│ • 价格锚点     │ • 视觉热区   │ • 感官描述            │
│ • 心理定价     │ • 金三角     │ • 故事化              │
│ • 价格带分布   │ • 首位效应   │ • 信任信号            │
│ • 套餐捆绑     │ • 分散利润款 │ • 价值暗示            │
│ • 尾数策略     │ • 去货币符号 │ • 场景化              │
├───────────────┴──────────────┴──────────────────────┤
│              套餐设计 (15%)                           │
│   捆绑策略 • 交叉补贴 • 推荐搭配 • 毛利组合            │
└─────────────────────────────────────────────────────┘
```

---

## 三、产品架构：MenuAI菜单优化引擎

```
┌─────────────────────────────────────────────────────┐
│              MenuAI 菜单优化引擎                        │
├─────────────────────────────────────────────────────┤
│                                                       │
│  输入：现有菜单 + 成本数据 + 餐厅定位 + 目标客单价        │
│                                                       │
│  ┌──────────────────────────────────────────────┐    │
│  │            AI分析层                            │    │
│  │                                                │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐   │    │
│  │  │ 出品分析  │ │ 毛利分析  │ │  竞品分析     │   │    │
│  │  │ (菜品结构)│ │ (成本利润)│ │  (竞争格局)   │   │    │
│  │  └──────────┘ └──────────┘ └──────────────┘   │    │
│  └──────────────────────────────────────────────┘    │
│                         ↓                             │
│  ┌──────────────────────────────────────────────┐    │
│  │           优化引擎层                            │    │
│  │                                                │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐   │    │
│  │  │ 定价优化  │ │ 排列优化  │ │  文案优化     │   │    │
│  │  │ (价格策略)│ │ (视觉布局)│ │  (感官营销)   │   │    │
│  │  └──────────┘ └──────────┘ └──────────────┘   │    │
│  │                                                │    │
│  │  ┌──────────┐                                  │    │
│  │  │ 套餐设计  │                                  │    │
│  │  │ (捆绑策略)│                                  │    │
│  │  └──────────┘                                  │    │
│  └──────────────────────────────────────────────┘    │
│                         ↓                             │
│  输出：优化后菜单 + 优化报告 + 预期效果分析              │
└─────────────────────────────────────────────────────┘
```

---

## 四、核心Java实现

### 4.1 智能定价引擎

```java
@Service
public class MenuPricingEngine {
    
    private final ChatClient chatClient;
    
    /**
     * 智能定价优化
     * 
     * 定价策略组合：
     * 1. 价格锚点：人为设置高价菜
     * 2. 心理定价：¥58 vs ¥60
     * 3. 价格带分布：引流款/利润款/形象款的比例
     * 4. 套餐定价：比单点合计便宜10%-15%
     * 5. 尾数策略：8/9结尾
     */
    public PricingOptimization optimizePricing(List<MenuItem> menuItems,
                                                RestaurantProfile profile,
                                                PricingGoals goals) {
        
        // Step 1: 计算每道菜的成本和毛利率
        List<MenuItemWithMargin> itemsWithMargin = menuItems.stream()
            .map(item -> {
                double cost = item.getFoodCost();
                double currentMargin = (item.getPrice() - cost) / item.getPrice();
                return new MenuItemWithMargin(item, cost, currentMargin);
            })
            .collect(Collectors.toList());
        
        // Step 2: 菜品分类（引流款/利润款/形象款）
        Map<MenuItemRole, List<MenuItemWithMargin>> categorized = 
            categorizeItems(itemsWithMargin, profile);
        
        // Step 3: LLM定价建议
        String pricingAdvice = generatePricingAdvice(
            categorized, profile, goals
        );
        
        // Step 4: 逐项优化定价
        List<PricedItem> optimizedItems = new ArrayList<>();
        
        for (MenuItemWithMargin item : itemsWithMargin) {
            MenuItemRole role = determineRole(item, categorized);
            
            // 计算最优价格区间
            PriceRange optimalRange = calculateOptimalRange(
                item, role, profile, goals
            );
            
            // 应用心理学定价
            double optimizedPrice = applyPsychologicalPricing(
                optimalRange.getOptimal(), item, profile
            );
            
            optimizedItems.add(new PricedItem(
                item.getMenuItem(),
                item.getCurrentPrice(),
                optimizedPrice,
                item.getCost(),
                (optimizedPrice - item.getCost()) / optimizedPrice,
                role,
                String.format("推荐理由：%s", generatePriceReason(
                    item, optimizedPrice, role, profile))
            ));
        }
        
        // Step 5: 验证优化效果
        PricingImpact impact = calculatePricingImpact(
            menuItems, optimizedItems, profile
        );
        
        return PricingOptimization.builder()
            .optimizedItems(optimizedItems)
            .strategyDescription(pricingAdvice)
            .expectedImpact(impact)
            .priceAnchors(generatePriceAnchors(optimizedItems))
            .bundleSuggestions(generateBundleSuggestions(optimizedItems, profile))
            .build();
    }
    
    /**
     * 菜品角色分类
     * 
     * 引流款（Traffic）：价格低/毛利低，吸引顾客进店
     * 利润款（Profit）：毛利高，是利润核心
     * 形象款（Image）：价格高/品质好，提升品牌档次，兼做价格锚点
     */
    private Map<MenuItemRole, List<MenuItemWithMargin>> categorizeItems(
            List<MenuItemWithMargin> items, RestaurantProfile profile) {
        
        List<MenuItemWithMargin> sorted = items.stream()
            .sorted(Comparator.comparingDouble(MenuItemWithMargin::getPrice))
            .collect(Collectors.toList());
        
        int size = sorted.size();
        int trafficEnd = (int) (size * 0.3);   // 前30%
        int profitEnd = (int) (size * 0.85);    // 中间55%
        
        Map<MenuItemRole, List<MenuItemWithMargin>> result = new HashMap<>();
        result.put(MenuItemRole.TRAFFIC, 
            sorted.subList(0, trafficEnd));
        result.put(MenuItemRole.PROFIT, 
            sorted.subList(trafficEnd, profitEnd));
        result.put(MenuItemRole.IMAGE, 
            sorted.subList(profitEnd, size));
        
        return result;
    }
    
    /**
     * 心理学定价策略
     */
    private double applyPsychologicalPricing(double basePrice,
                                              MenuItemWithMargin item,
                                              RestaurantProfile profile) {
        
        // 策略1: 尾数效应
        // ¥20-100区间 → 尾数用8或9
        // ¥100+ → 尾数用8
        if (basePrice < 10) {
            return roundToNearest9(basePrice);  // ¥9.9
        } else if (basePrice < 100) {
            return Math.floor(basePrice) + 0.8; // ¥58.8
        } else {
            return Math.floor(basePrice / 10) * 10 + 8; // ¥198
        }
    }
    
    private double roundToNearest9(double price) {
        return Math.floor(price) + 0.9;
    }
    
    /**
     * 价格锚点生成
     * 
     * 经典策略：设置1-2个显著高于其他菜品的"锚点菜"
     * 作用：让顾客的参考价格基准上移
     * 
     * 例如：设置一道¥688的5A和牛，顾客会觉得¥188的普通牛排"不贵"
     */
    private List<PriceAnchor> generatePriceAnchors(List<PricedItem> items) {
        
        // 找到最贵的3道菜
        List<PricedItem> top3 = items.stream()
            .sorted(Comparator.comparingDouble(PricedItem::getOptimizedPrice).reversed())
            .limit(3)
            .collect(Collectors.toList());
        
        // 检查是否有足够的锚定效果（最贵/次贵 > 2倍）
        if (top3.size() >= 2) {
            double ratio = top3.get(0).getOptimizedPrice() 
                         / top3.get(1).getOptimizedPrice();
            
            if (ratio < 1.8) {
                // 锚定效果不够，建议提高形象款价格
                return List.of(PriceAnchor.builder()
                    .anchorItem(top3.get(0))
                    .suggestedIncrease(top3.get(0).getOptimizedPrice() * 0.3)
                    .reason("锚定效果不足（最贵/次贵=%.1f倍，建议>2倍），提高形象款价格".formatted(ratio))
                    .build());
            }
        }
        
        return Collections.emptyList();
    }
}
```

### 4.2 菜单排列优化

```java
@Service
public class MenuLayoutOptimizer {
    
    private final ChatClient chatClient;
    
    /**
     * 菜单排列优化
     * 
     * 基于"金三角"理论和视觉热区研究
     * 
     * 顾客扫描菜单的视觉路径：
     * 
     *    ① → ② → ③
     *    ↓         ↓
     *    ④   ⑥    ⑤
     *    ↓         ↓
     *    ⑦ ← ⑧ ← ⑨
     * 
     * ①=黄金位置（顾客第一眼看到的地方）
     * ②=右上（次热区）
     * ⑤=右下（第三热区）
     */
    public LayoutOptimization optimizeLayout(List<PricedItem> pricedItems,
                                              MenuFormat format,
                                              LayoutGoals goals) {
        
        // 根据菜单格式生成热区映射
        HotZoneMap hotZoneMap = generateHotZoneMap(format);
        
        // 使用LLM推荐最优排列
        String layoutJson = chatClient.prompt()
            .system("""
                你是菜单设计大师，精通视觉心理学和餐厅运营。
                
                菜单排列原则：
                1. 金三角位置（顶部中右）：放利润款
                2. 首位效应（第一道菜的影响巨大）：放招牌菜/利润款
                3. 视线路径：顾客按"Z字形"或"倒N字形"扫描
                4. 分组策略：同类菜集中，方便比较和选择
                5. 去货币符号：不写¥符号，弱化价格敏感度
                6. 分散策略：不要把最贵的菜都放在一起
                
                请以JSON格式输出每道菜的推荐位置：
                [{"itemName": "...", "page": 1, "position": {"row": 1, "col": 1}, "reason": "..."}]
                """)
            .user("""
                菜单格式：%s（%d页）
                菜单品类：%s
                菜品总数：%d道
                
                引流款（%d道）：
                %s
                
                利润款（%d道）：
                %s
                
                形象款（%d道）：
                %s
                
                目标：提升利润款点击率，引导点单决策。
                """.formatted(
                    format.getType(), format.getPages(),
                    goals.getCuisine(),
                    pricedItems.size(),
                    countByRole(pricedItems, MenuItemRole.TRAFFIC),
                    formatItemsByRole(pricedItems, MenuItemRole.TRAFFIC),
                    countByRole(pricedItems, MenuItemRole.PROFIT),
                    formatItemsByRole(pricedItems, MenuItemRole.PROFIT),
                    countByRole(pricedItems, MenuItemRole.IMAGE),
                    formatItemsByRole(pricedItems, MenuItemRole.IMAGE)
                ))
            .call()
            .content();
        
        return parseLayoutOptimization(layoutJson, pricedItems, hotZoneMap);
    }
    
    /**
     * 菜单热区映射
     * 不同格式（单页/双页/折页）的视觉热区不同
     */
    private HotZoneMap generateHotZoneMap(MenuFormat format) {
        
        return switch (format.getType()) {
            case SINGLE_PAGE -> HotZoneMap.builder()
                .primaryZone(Position.of(0, 0))       // 左上角 = 黄金位置
                .secondaryZone(Position.of(0, 1))      // 右上角 = 次热区
                .tertiaryZone(Position.of(1, 1))       // 右下 = 第三热区
                .coldZones(List.of(
                    Position.of(2, 0),                  // 左下角 = 冷区
                    Position.of(2, 1)                   // 中下 = 冷区
                ))
                .build();
                
            case DOUBLE_PAGE -> HotZoneMap.builder()
                .primaryZone(Position.of(0, 0))        // 左页左上
                .secondaryZone(Position.of(0, 2))       // 右页右上
                .tertiaryZone(Position.of(1, 1))        // 中间位置
                .coldZones(List.of(
                    Position.of(2, 0),                   // 左页左下
                    Position.of(2, 2)                    // 右页右下
                ))
                .build();
                
            case TRIFOLD -> HotZoneMap.builder()
                .primaryZone(Position.of(0, 0))
                .secondaryZone(Position.of(0, 1))
                .tertiaryZone(Position.of(1, 0))
                .build();
        };
    }
}
```

### 4.3 菜品文案AI生成

```java
@Service
public class MenuCopyGenerator {
    
    private final ChatClient chatClient;
    
    /**
     * 为每道菜生成吸引力强的文案
     * 
     * 文案策略：
     * 1. 感官描述：让顾客"看"到、"闻"到、"尝"到
     * 2. 产地故事：增加价值感和信任
     * 3. 工艺描述：强调手工/匠心
     * 4. 场景触发："最适合配一杯..."
     * 5. 社交证明："本店招牌""回头率最高"
     */
    public Map<String, String> generateDescriptions(List<MenuItem> items,
                                                      RestaurantProfile profile) {
        
        Map<String, String> descriptions = new LinkedHashMap<>();
        
        for (MenuItem item : items) {
            String description = chatClient.prompt()
                .system("""
                    你是米其林餐厅的菜单文案大师。为菜品撰写吸引力描述。
                    
                    文案规则：
                    1. 15-30字，短而有力
                    2. 必须包含：1个感官词 + 1个产地/工艺词 + 1个价值词
                    3. 避免枯燥的原料罗列
                    4. 用逗号/顿号分隔，不要长句
                    5. 根据菜品角色调整风格：
                       - 利润款：侧重"独特体验""匠心工艺"
                       - 引流款：侧重"经典""人气"
                       - 形象款：侧重"稀缺""顶级的"
                    
                    示例：
                    × 三文鱼刺身，新鲜
                    ✓ 挪威空运三文鱼，厚切一口鲜甜
                    
                    × 红烧肉，好吃
                    ✓ 慢炖4小时黑毛猪，肥而不腻入口即化
                    """)
                .user("""
                    餐厅类型：%s
                    菜品名称：%s
                    主要食材：%s
                    烹饪工艺：%s
                    菜品价格：¥%.0f
                    菜品角色：%s
                    
                    请生成3个不同风格的描述供选择。
                    JSON: ["描述1", "描述2", "描述3"]
                    """.formatted(
                        profile.getCuisine(),
                        item.getName(),
                        item.getIngredients(),
                        item.getCookingMethod(),
                        item.getPrice(),
                        item.getRole()
                    ))
                .call()
                .content();
            
            List<String> options = parseDescriptions(description);
            descriptions.put(item.getName(), options.get(0));
        }
        
        return descriptions;
    }
    
    /**
     * 为菜品生成"信任标签"
     * 如：🏆 本店招牌 | ⭐ 回头率TOP3 | 🔥 新品尝鲜 | 🌿 有机认证
     */
    public Map<String, List<String>> generateTrustBadges(List<MenuItem> items) {
        
        String badgesJson = chatClient.prompt()
            .system("""
                为菜品生成"信任标签"（emoji + 2-4字）。
                
                标签类型：
                - 🏆 本店招牌、主厨推荐
                - ⭐ 人气TOP、回头客最爱
                - 🔥 季节限定、新品尝鲜
                - 🌿 有机认证、农场直供
                - 🥇 获奖菜品
                - 💪 高蛋白、低脂健康
                
                JSON: {"菜品名": ["🥇 获奖菜品", "🏆 主厨推荐"]}
                """)
            .user("菜品列表：%s".formatted(
                items.stream()
                    .map(MenuItem::getName)
                    .collect(Collectors.joining(", "))
            ))
            .call()
            .content();
        
        return parseBadges(badgesJson);
    }
}
```

### 4.4 套餐智能设计

```java
@Service
public class BundleDesigner {
    
    /**
     * 套餐智能设计
     * 
     * 策略：
     * 1. 组合高毛利+低毛利菜品（整体毛利优化）
     * 2. 比单买便宜10-15%（让顾客觉得"划算"）
     * 3. 用引流款带动利润款
     * 4. 设计"比较套餐"：加¥X元升级
     */
    public List<MenuBundle> designBundles(List<PricedItem> items,
                                           RestaurantProfile profile) {
        
        List<MenuBundle> bundles = new ArrayList<>();
        
        // 套餐1: 单人招牌套餐
        bundles.add(designSingleBundle(items, "单人招牌"));
        
        // 套餐2: 双人分享套餐
        bundles.add(designCoupleBundle(items, "双人分享"));
        
        // 套餐3: 家庭套餐
        bundles.add(designFamilyBundle(items, "家庭欢聚"));
        
        // 套餐4: 加价升级策略
        bundles.add(designUpgradeBundle(items, "超值升级"));
        
        return bundles;
    }
    
    private MenuBundle designSingleBundle(List<PricedItem> items, String name) {
        
        String bundleJson = chatClient.prompt()
            .system("""
                设计%{cuisine}单人套餐。选择前菜+主菜+饮品(各1道)。
                
                选择策略：
                1. 搭配合理（味道不冲突）
                2. 整体毛利率>40%
                3. 总价比单买便宜12%
                4. 包含1道利润款
                
                JSON: {
                  "name": "套餐名",
                  "items": ["菜品1", "菜品2", "菜品3"],
                  "singleTotal": 150.0,
                  "bundlePrice": 132.0,
                  "savings": 18.0,
                  "marginRate": 0.42,
                  "description": "套餐说明文案"
                }
                """)
            .user("菜品列表：%s".formatted(formatItemsForPrompt(items)))
            .call()
            .content();
        
        return parseBundle(bundleJson);
    }
}
```

---

## 五、优化效果验证

### 某日料店优化前后对比（上线30天后）

| 指标 | 优化前 | 优化后 | 变化 |
|------|--------|--------|------|
| 客单价 | ¥122 | ¥148 | **+21.3%** |
| 利润款点击率 | 23% | 41% | **+78%** |
| 套餐点单率 | 18% | 35% | **+94%** |
| 综合毛利率 | 58% | 67% | **+9%** |
| 月营收 | ¥31万 | ¥38万 | **+22.6%** |

---

## 六、商业模式

| 版本 | 价格 | 菜品数 |
|------|------|--------|
| 免费版 | ¥0 | 20道/月 |
| 轻量版 | ¥99/月 | 100道 |
| 专业版 | ¥299/月 | 500道 + 竞品分析 |
| 连锁版 | ¥999/月起 | 无限 + 多门店 |

---

> **下一篇预告**：《实体店AI点评回复——客人写100字差评，AI生成300字真诚回复》，大众点评美团上的差评回复比好评更重要，AI帮实体店把每条差评转化成为品牌加分的机会。
