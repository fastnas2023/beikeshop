# BeikeShop 模板插件开发指南

在 BeikeShop 中，您可以通过**插件（Plugin）**的方式来开发模板（主题或页面组件）。插件化的模板开发能够保证在不修改系统核心代码的情况下，安全地替换前台界面或在页面指定位置插入自定义的 HTML/Blade 视图。

---

## 1. 模板插件的类型

BeikeShop 中的“模板插件”主要有以下两种实现形式：

1. **局部替换/注入型模板插件**（推荐）
   通过系统内置的 `Hook` 机制，在现有页面的特定埋点（例如头部、底部、商品详情页）注入您的自定义 Blade 模板代码。
2. **全局主题插件 (Theme Plugin)**
   通过定义 `config.json` 中 `"type": "theme"`，将插件识别为全局主题，并利用 Laravel 的视图命名空间覆盖系统的默认视图文件。

---

## 2. 插件的基本目录结构

在 `plugins/` 目录下创建一个新的插件文件夹，例如 `MyThemePlugin`。基本的目录结构如下：

```text
plugins/
└── MyThemePlugin/
    ├── Bootstrap.php       # 插件的启动文件，核心 Hook 注册逻辑在这里
    ├── config.json         # 插件配置信息（必需）
    ├── columns.php         # （可选）后台设置面板的表单字段配置
    ├── Routes/             # （可选）自定义路由
    │   ├── admin.php       
    │   └── shop.php        
    ├── Static/             # （可选）前端静态资源 (CSS, JS, Images)
    │   ├── css/
    │   ├── js/
    │   └── image/
    └── Views/              # 自定义的 Blade 模板目录
        ├── admin/          # 注入到后台的视图
        └── shop/           # 注入到前台的视图
```

---

## 3. 开发步骤详解

### 步骤 1: 编写 `config.json`

这是插件的身份证，系统依赖此文件识别插件。如果是纯模板/主题，请将 `"type"` 设置为 `"theme"` 或 `"feature"`。

```json
{
    "code": "my_theme_plugin",
    "name": {
        "zh_cn": "我的自定义模板插件",
        "en": "My Custom Theme Plugin"
    },
    "description": {
        "zh_cn": "这是一个用于演示的模板插件，修改了首页头部并增加了自定义组件",
        "en": "A demo theme plugin."
    },
    "type": "theme",
    "version": "v1.0.0",
    "icon": "/image/logo.png",
    "author": {
        "name": "Your Name",
        "email": "your.email@example.com"
    }
}
```

### 步骤 2: 编写 `Bootstrap.php` (注册 Hook 注入视图)

`Bootstrap.php` 是插件的入口，系统在启动时会调用其 `boot()` 方法。我们可以使用 `add_hook_blade` 和 `add_hook_filter` 函数来注入或修改模板渲染。

> **提示：** BeikeShop 前台模板中大量使用了 `@hook('xxx')` 和 `@hookwrapper('xxx')` 标签，这就是您可以挂载模板的地方。

```php
<?php
namespace Plugin\MyThemePlugin;

class Bootstrap
{
    public function boot()
    {
        $this->modifyHeader();
        $this->modifyProductDetail();
    }

    /**
     * 演示：修改前台全局 header 的局部模板
     */
    private function modifyHeader()
    {
        // 在 header 的 logo 后面追加一段自定义的视图
        add_hook_blade('header.menu.logo', function ($callback, $output, $data) {
            // 渲染插件目录下的 Views/shop/custom_logo_text.blade.php
            $customView = view('MyThemePlugin::shop.custom_logo_text', $data)->render();
            
            // $output 是原本系统要输出的内容
            return $output . $customView; 
        });
    }

    /**
     * 演示：修改商品详情页的模板及数据
     */
    private function modifyProductDetail()
    {
        // 1. 通过模板 hook 在商品名称前加一个 Badge 视图
        add_hook_blade('product.detail.name', function ($callback, $output, $data) {
            $badgeView = view('MyThemePlugin::shop.badge')->render();
            return $badgeView . $output;
        });

        // 2. 通过模板 hook 在"立即购买"按钮后追加一个自定义按钮
        add_hook_blade('product.detail.buy.after', function ($callback, $output, $data) {
            $btnView = view('MyThemePlugin::shop.extra_button')->render();
            return $output . $btnView;
        });
    }
}
```

### 步骤 3: 编写 Blade 模板文件

在插件中创建对应的 Blade 视图文件。注意，引用插件视图时，命名空间格式为 `插件文件夹名::路径`，例如 `MyThemePlugin::shop.badge` 对应 `plugins/MyThemePlugin/Views/shop/badge.blade.php`。

**`plugins/MyThemePlugin/Views/shop/badge.blade.php`**
```html
<span class="badge" style="background-color: #FF4D00; color: #fff; margin-right: 8px;">
    新模板专属
</span>
```

**`plugins/MyThemePlugin/Views/shop/extra_button.blade.php`**
```html
<button class="btn btn-outline-primary ms-2" onclick="alert('这是模板插件新增的按钮！')">
    <i class="bi bi-star"></i> 收藏
</button>
```

### 步骤 4: 处理静态资源 (CSS / JS)

如果您需要为模板添加额外的 CSS 或 JS 文件：
1. 将它们放在 `plugins/MyThemePlugin/Static/css/` 或 `plugins/MyThemePlugin/Static/js/` 目录下。
2. 可以在 `Bootstrap.php` 中利用 Hook 将资源文件注入到系统的 `<head>` 或底部。

```php
add_hook_blade('layout.header.css', function ($callback, $output, $data) {
    $cssPath = asset('plugin/MyThemePlugin/css/style.css');
    return $output . '<link rel="stylesheet" href="' . $cssPath . '">';
});
```
*(注意：需要确保系统的静态资源发布机制将 Plugin 的 Static 目录软链接或复制到了 public 目录下，具体参考系统安装与插件管理后台。)*

---

## 4. 全局主题覆盖 (Advanced)

如果您的插件不仅仅是局部注入，而是想**完全替换**前台的默认主题视图，可以通过设置 `config.json` 的 `"type": "theme"`。

在 BeikeShop 中，开启主题插件后，可以在系统的 **“后台管理 -> 系统设置 -> 基础设置 -> 默认主题”** 中选择该插件的标识（如 `my_theme_plugin`）。
系统底层的 `ShopServiceProvider` 会自动将 `themes/你的主题名` 或对应的主题路径优先级调高，从而覆盖默认的视图。

通常对于复杂的全局主题开发，建议直接在根目录的 `themes/自定义主题名` 目录下按照默认主题 `themes/default/` 的结构复制一份进行修改，并在后台设置中切换主题即可，这也是 BeikeShop 推荐的全局模板开发方式。而**插件式的模板开发更适合“功能+局部 UI 增强”**的场景。

---

## 5. 安装与启用

1. 在本地开发完成后，将 `MyThemePlugin` 压缩为 `MyThemePlugin.zip`。
2. 登录 BeikeShop 后台，进入 **插件管理 -> 上传插件**。
3. 上传后，在列表中找到“我的自定义模板插件”，点击 **“安装”** 并 **“启用”**。
4. 刷新商城前台，即可看到通过 Hook 注入的自定义模板内容！
