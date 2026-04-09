# BeikeShop Code Wiki

## 1. 项目整体架构 (Project Architecture)

BeikeShop 是一款基于 **PHP 8.1+** 和 **Laravel 10** 框架开发的开源跨境电商系统。系统整体采用了 **MVC（模型-视图-控制器）** 架构，并且引入了**模块化 (Modular)** 和**事件驱动 (Event-driven)** 的设计理念。

区别于传统的 Laravel 项目结构，BeikeShop 将其核心业务逻辑从默认的 `app/` 目录中抽离，独立封装在 `beike/` 目录下。这种设计使得业务代码与框架代码解耦，便于系统的维护与后续升级。此外，系统通过内置的 **Hook (钩子)** 和 **Plugin (插件)** 机制，允许开发者在不修改核心源代码的情况下（非侵入式）进行二次开发和功能扩展。

### 核心技术栈
*   **后端框架**: PHP 8.1+ / Laravel 10
*   **前端渲染**: Blade Templates, Vue.js, Bootstrap, jQuery, Sass
*   **数据库**: MySQL 5.7+ / PostgreSQL
*   **缓存与队列**: Redis (通过 Laravel Horizon/Queue)
*   **扩展机制**: Hook (事件/钩子引擎) / Plugin (微内核插件化系统)
*   **部署**: Docker 容器化部署 / 传统 LNMP/LAMP 部署

---

## 2. 主要模块职责 (Main Modules)

系统核心代码集中在 [beike](file:///workspace/beike) 目录，各个主要子模块的职责如下：

*   **[Admin](file:///workspace/beike/Admin)**: 负责后台管理系统的核心逻辑。包含后台的控制器 (Controllers)、请求验证 (Requests)、API资源 (Resources) 以及专供后台使用的 Blade 视图组件。
*   **[Shop](file:///workspace/beike/Shop)**: 负责前台商城系统（即面向消费者的 C 端）。包含用户账户 (Account)、购物车 (Cart)、结账 (Checkout) 等业务流程的控制器、视图以及专属服务。
*   **[AdminAPI](file:///workspace/beike/AdminAPI)**: 为后台提供的 RESTful API 接口层，支持前后端分离或移动端 App 的后台数据交互。
*   **[Models](file:///workspace/beike/Models)**: 基于 Eloquent ORM 的数据模型层，映射数据库表结构（如 `Product`, `Order`, `Customer`, `Category` 等）。
*   **[Repositories](file:///workspace/beike/Repositories)**: 数据仓库层。封装了底层数据库查询逻辑，使得 Controller 层与底层数据解耦，提高代码的复用性（例如 `ProductRepo`, `OrderRepo`）。
*   **[Services](file:///workspace/beike/Services)**: 业务逻辑层。处理复杂的业务规则（例如 `CheckoutService`, `PaymentService`, `CartService`）。
*   **[Hook](file:///workspace/beike/Hook)**: 钩子与事件引擎。允许在系统生命周期的特定节点挂载自定义逻辑或修改视图输出，是实现非侵入式开发的核心模块。
*   **[Plugin](file:///workspace/beike/Plugin)**: 插件管理模块。负责读取、验证、解析并加载 `plugins/` 目录下的第三方插件。
*   **[Installer](file:///workspace/beike/Installer)**: 系统安装向导。负责在首次部署时引导用户进行环境检测、数据库配置、基础数据填充等操作。
*   **[plugins](file:///workspace/plugins)**: 存放已安装的第三方扩展插件（例如 `Bestseller`, `Paypal`, `Stripe`, `Wtp`, `Youdao` 等）。
*   **[themes](file:///workspace/themes)**: 存放系统前台的主题模板（如 `default` 主题），支持自定义覆盖渲染。

---

## 3. 关键类与函数说明 (Key Classes & Functions)

### 3.1 Hook 系统 (`Beike\Hook\Hook`)
路径：[Hook.php](file:///workspace/beike/Hook/Hook.php)
*   **职责**: BeikeShop 的核心扩展引擎，负责注册和触发钩子事件。
*   **`listen(string $hook, $function, $priority = null)`**: 监听（订阅）指定的 Hook 事件，允许传入回调函数或类方法。
*   **`getHook(string $hook, array $params = [], callable $callback = null, string $htmlContent = '')`**: 触发并执行某个 Hook。通常埋点在控制器或视图中，将执行权临时交给监听了该 Hook 的插件代码。
*   **`getWrapper(string $hook, ...)`**: 包装型 Hook，用于对系统输出的一段 HTML 内容进行拦截、修改并返回新内容。

### 3.2 Plugin 管理器 (`Beike\Plugin\Manager`)
路径：[Manager.php](file:///workspace/beike/Plugin/Manager.php)
*   **职责**: 解析和管理系统中的插件包。
*   **`getPlugins()`**: 扫描并解析 `plugins/` 目录，返回所有安装的插件集合。
*   **`getEnabledBootstraps()`**: 获取所有已开启插件的引导文件 (`bootstrap.php`)，并在框架启动时加载它们。
*   **`import(UploadedFile $file)`**: 负责在后台上传插件 ZIP 包，包含安全检查、解压以及防止 ZIP 炸弹和目录遍历的安全防御逻辑。

### 3.3 商城基础服务商 (`Beike\Shop\Providers\ShopServiceProvider`)
路径：[ShopServiceProvider.php](file:///workspace/beike/Shop/Providers/ShopServiceProvider.php)
*   **职责**: 初始化前台商城的运行环境。
*   **`boot()`**: 框架启动入口。在这里加载路由 (`shop.php`)、加载邮件配置、注册前端客户会话守卫 (Guard `shop_customer`)、设置文件上传磁盘，并调用 `loadThemeViewPath()` 挂载当前启用的前台主题视图目录。

### 3.4 结账/购物车服务 (`Beike\Shop\Services\CheckoutService` / `CartService`)
路径：[CheckoutService.php](file:///workspace/beike/Shop/Services/CheckoutService.php)
*   **职责**: 封装了商城最为复杂的购物车结算、订单生成、价格计算以及库存扣减逻辑。通常配合 `OrderTotalService` 动态计算包含运费、税费和优惠折扣在内的最终价格。

---

## 4. 依赖关系 (Dependencies)

项目依赖主要通过 `composer.json` 进行管理，核心扩展包如下：

*   **laravel/framework (^10.0)**: 核心框架。
*   **doctrine/dbal (^3.7)**: 用于高级数据库字段修改及操作。
*   **intervention/image (^2.7)**: 用于商品图片、横幅等图片的裁剪与处理。
*   **spatie/laravel-permission (^5.5)**: 提供后台管理系统的 RBAC（基于角色的访问控制）权限管理。
*   **tymon/jwt-auth (^2.0)**: 用于 API 的 JSON Web Token 认证机制。
*   **srmklive/paypal (^3.0)** / **stripe/stripe-php (^8.8)**: 用于集成 PayPal 和 Stripe 国际支付网关。
*   **w7corp/easywechat (^6.7)**: 用于集成微信支付及微信生态相关功能。
*   **zanysoft/laravel-zip (^2.0)**: 处理系统安装、插件包解压以及系统备份的压缩操作。

---

## 5. 项目运行方式 (How to Run)

### 5.1 环境要求
*   **操作系统**: Ubuntu 22+ / CentOS 8.5
*   **PHP**: 8.2 (需开启 BCMath, cURL, DOM, Fileinfo, JSON, OpenSSL, PDO 等扩展)
*   **数据库**: MySQL 5.7+
*   **Web 服务器**: Nginx 1.10+ / Apache 2.4+

### 5.2 源码安装步骤

1. **克隆项目并安装依赖**:
   ```bash
   git clone https://github.com/beikeshop/beikeshop.git
   cd beikeshop
   composer install
   ```

2. **配置环境文件**:
   ```bash
   cp .env.example .env
   # 根据实际数据库信息修改 .env 文件中的 DB_* 环境变量
   ```

3. **编译前端静态资源**:
   ```bash
   npm install
   npm run prod
   ```

4. **配置 Web 服务器并完成安装**:
   * 将 Web 服务器（如 Nginx/Apache）的文档根目录（Document Root）指向项目的 `public` 目录。
   * 确保 `storage` 和 `bootstrap/cache` 目录具有写入权限。
   * 在浏览器中访问站点域名，系统将自动跳转至内置的 **Web 安装向导** (`/installer`)，按照界面提示配置数据库并初始化管理员账号。

### 5.3 Docker 安装步骤 (推荐)

BeikeShop 提供了官方的 Docker 环境支持，可以快速拉起项目：

1. **克隆 Docker 编排仓库**:
   ```bash
   git clone git@gitee.com:beikeshop/docker.git
   cd docker
   mkdir www
   ```

2. **配置与启动**:
   ```bash
   cp env.example .env
   docker compose up -d
   ```
   随后将源码放置于 `www` 目录，通过浏览器访问本地地址进行向导安装。

### 5.4 本地开发启动 (Laravel Artisan)
如果仅用于本地快速预览，在完成 `composer install` 与 `.env` 配置后，可直接使用内置服务：
```bash
php artisan serve
```
访问 `http://localhost:8000` 即可进入系统。

---
> **Note**: 有关更详细的二开指南或 API 对接，建议查阅项目内的 [README.md](file:///workspace/README.md) 或访问 [BeikeShop 官方文档](https://docs.beikeshop.com)。
