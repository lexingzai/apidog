## Api Watch Dog
一个 [Hyperf](https://github.com/hyperf/hyperf) 框架的 Api 参数校验及 swagger 文档生成组件

1.  根据注解自动进行Api参数的校验, 业务代码更纯粹.
2.  根据注解自动生成Swagger文档, 让接口文档维护更省心.

> 在 1.2 版本后, 本扩展移除了内部自定义的验证器, 只保留的 hyperf 原生验证器, 以保持验证规则的统一

旧版本文档 [查看](./README_OLD.md)

## 安装

```
composer require daodao97/apidog
```
## 使用

#### 1. 发布配置文件

```bash
php bin/hyperf.php vendor:publish daodao97/apidog

# hyperf/validation 的依赖发布

php bin/hyperf.php vendor:publish hyperf/translation

php bin/hyperf.php vendor:publish hyperf/validation
```

### 2. 修改配置文件

> 注意 与1.2及之前的版本相比, 配置文件结构及文件名 略有不同
> 
> (1) 配置文件结构的优化, 增加了swagger外的整体配置
>
> (2) 配置文件的名称 由 swagger.php 改为 apidog.php

根据需求修改 `config/autoload/apidog.php`

```php
<?php

return [
    // enable false 将不会生成 swagger 文件
    'enable' => env('APP_ENV') !== 'production',
    // swagger 配置的输出文件
    // 当你有多个 http server 时, 可以在输出文件的名称中增加 {server} 字面变量
    // 比如 /public/swagger/swagger_{server}.json
    'output_file' => BASE_PATH . '/public/swagger/swagger.json',
    // 忽略的hook, 非必须 用于忽略符合条件的接口, 将不会输出到上定义的文件中
    'ignore' => function ($controller, $action) {
        return false;
    },
    // 自定义验证器错误码、错误描述字段
    'error_code' => 400,
    'http_status_code' => 400,
    'field_error_code' => 'code',
    'field_error_message' => 'message',
    // swagger 的基础配置
    'swagger' => [
        'swagger' => '2.0',
        'info' => [
            'description' => 'hyperf swagger api desc',
            'version' => '1.0.0',
            'title' => 'HYPERF API DOC',
        ],
        'host' => 'hyperf2.test',
        'schemes' => ['http'],
    ],
];
```

### 3. 启用 Api参数校验中间件

```php
// config/autoload/middlewares.php

Hyperf\Apidog\Middleware\ApiValidationMiddleware::class
```

### 4. 校验规则的定义

规则列表参见 [hyperf/validation 文档](https://hyperf.wiki/#/zh-cn/validation?id=%e9%aa%8c%e8%af%81%e8%a7%84%e5%88%99)

更详细的规则支持列表可以参考 [laravel/validation 文档](https://learnku.com/docs/laravel/6.x/validation/5144#c58a91)

扩展在原生的基础上进行了封装, 支持方便的进行 `自定义校验` 和 `控制器回调校验`

## 实现思路

api参数的自动校验: 通过中间件拦截 http 请求, 根据注解中的参数定义, 通过 `valiation` 自动验证和过滤, 如果验证失败, 则拦截请求. 其中`valiation` 包含 规则校验, 参数过滤, 自定义校验 三部分. 

swagger文档生成: 在`php bin/hyperf.php start` 启动 `http-server` 时, 通过监听 `BootAppConfListener` 事件, 扫码控制器注解, 通过注解中的 访问类型, 参数格式, 返回类型 等, 自动组装 `swagger.json` 结构, 最后输出到 `config/autoload/apidog.php` 定义的文件路径中

## 支持的注解 

#### Api类型
`GetApi`, `PostApi`, `PutApi`, `DeleteApi`

### 参数类型
`Header`, `Quyer`, `Body`, `FormData`, `Path`

### 其他
`ApiController`, `ApiResponse`, `ApiVersion`, `ApiServer`, `ApiDefinitions`, `ApiDefinition`

```php
/**
 * @ApiVersion(version="v1")
 * @ApiServer(name="http")
 */
class UserController {} 
```

`ApiServer` 当你在 `config/autoload.php/server.php servers` 中配置了多个 `http` 服务时, 如果想不同服务生成不同的`swagger.json` 可以在控制器中增加此注解.

`ApiVersion` 当你的统一个接口存在不同版本时, 可以使用此注解, 路由注册时会为每个木有增加版本号, 如上方代码注册的实际路由为 `/v1/user/***`

`ApiDefinition` 定义一个 `Definition`，用于Response的复用。 *swagger* 的difinition是以引用的方式来嵌套的，如果需要嵌套另外一个(值为object类型就需要嵌套了)，可以指定具体 `properties` 中的 `$ref` 属性

`ApiDefinitions` 定义一个组`Definition`

`ApiResponse` 响应体的`schema`支持为key设置简介. `$ref` 属性可以引用 `ApiDefinition` 定义好的结构(该属性优先级最高)
```php
@ApiResponse(code="0", description="删除成功", schema={"id|这里是ID":1})
@ApiResponse(code="0", description="删除成功", schema={"$ref": "ExampleResponse"})
```

具体使用方式参见下方样例

## 样例

```php
<?php
declare(strict_types=1);
namespace App\Controller;

use Hyperf\Apidog\Annotation\ApiController;
use Hyperf\Apidog\Annotation\ApiResponse;
use Hyperf\Apidog\Annotation\ApiVersion;
use Hyperf\Apidog\Annotation\Body;
use Hyperf\Apidog\Annotation\DeleteApi;
use Hyperf\Apidog\Annotation\FormData;
use Hyperf\Apidog\Annotation\GetApi;
use Hyperf\Apidog\Annotation\Header;
use Hyperf\Apidog\Annotation\PostApi;
use Hyperf\Apidog\Annotation\Query;
use Hyperf\Apidog\Annotation\ApiDefinitions;
use Hyperf\Apidog\Annotation\ApiDefinition;
use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\Utils\ApplicationContext;

/**
 * @ApiVersion(version="v1")
 * @ApiController(tag="用户管理", description="用户的新增/修改/删除接口")
 * @ApiDefinitions({
 *  @ApiDefinition(name="UserOkResponse", properties={
 *     "code|响应码": 200,
 *     "msg|响应信息": "ok",
 *     "data|响应数据": {"$ref": "UserInfoData"}
 *  }),
 *  @ApiDefinition(name="UserInfoData", properties={
 *     "userInfo|用户数据": {"$ref": "UserInfoDetail"}
 *  }),
 *  @ApiDefinition(name="UserInfoDetail", properties={
 *     "id|用户ID": 1,
 *     "mobile|用户手机号": { "default": "13545321231", "type": "string" },
 *     "nickname|用户昵称": "nickname",
 *     "avatar": { "default": "avatar", "type": "string", "description": "用户头像" },
 *  })
 * })
 */
class UserController extends AbstractController
{
    /**
     * @Author 刀刀
     * @PostApi(path="/user", description="添加一个用户")
     * @Header(key="token|接口访问凭证", rule="required")
     * @FormData(key="name|名称", rule="required|max:10|cb_checkName")
     * @FormData(key="sex|年龄", rule="integer|in:0,1")
     * @FormData(key="file|文件", rule="file")
     * @ApiResponse(code="-1", description="参数错误")
     * @ApiResponse(code="0", description="创建成功", schema={"id":1})
     */
    public function add()
    {
        return [
            'code' => 0,
            'id' => 1,
        ];
    }

    // 自定义的校验方法 rule 中 cb_*** 方式调用
    public function checkName($attribute, $value)
    {
        if (in_black_list($value)) {
            return "拒绝添加 " . $value;
        }

        return true;
    }

    /**
     * 请注意 body 类型 rules 为数组类型
     * @DeleteApi(path="/user", description="删除用户")
     * @Body(rules={
     *     "id|用户id":"required|integer|max:10",
     *     "deepAssoc|深层关联":{
     *        "name_1|名称": "required|integer|max:20"
     *     },
     *     "deepUassoc|深层索引":{{
     *         "name_2|名称": "required|integer|max:20"
     *     }}
     * })
     * @ApiResponse(code="-1", description="参数错误")
     * @ApiResponse(code="0", description="删除成功", schema={"id":1})
     */
    public function delete()
    {
        $request = ApplicationContext::getContainer()->get(RequestInterface::class);
        $body = $request->getBody()->getContents();
        return [
            'code' => 0,
            'query' => $request->getQueryParams(),
            'body' => json_decode($body, true),
        ];
    }

    /**
     * @GetApi(path="/user", description="获取用户详情")
     * @Query(key="id", rule="required|integer|max:0")
     * @ApiResponse(code="-1", description="参数错误")
     * @ApiResponse(code="0", schema={"id":1,"name":"张三","age":1})
     */
    public function get()
    {
        return [
            'code' => 0,
            'id' => 1,
            'name' => '张三',
            'age' => 1,
        ];
    }

    /**
     * schema中可以指定$ref属性引用定义好的definition
     * @GetApi(path="/user/info", description="获取用户详情")
     * @Query(key="id", rule="required|integer|max:0")
     * @ApiResponse(code="-1", description="参数错误")
     * @ApiResponse(code="0", schema={"$ref": "UserOkResponse"})
     */
    public function info()
    {
        return [
            'code' => 0,
            'id' => 1,
            'name' => '张三',
            'age' => 1,
        ];
    }
}
```

## Swagger UI启动


如果使用的是docker环境，需在.env配置PHP_CONTAINER_IP参数.

```bash
PHP_CONTAINER_IP=php容器的ip地址
```

组件提供了一个快捷命令, 用来快速启动一个 `swagger ui`,docker环境需手动打开浏览器.

```bash
php bin/hyperf.php apidog:ui
```

![DSRQnj](https://cdn.jsdelivr.net/gh/daodao97/FigureBed@master/uPic/DSRQnj.png)

## Swagger展示

![swagger](http://tva1.sinaimg.cn/large/007X8olVly1g6j91o6xroj31k10u079l.jpg)

## 更新日志
- 20200904
    - 增加 `ApiDefinitions` 与 `ApiDefinition` 注解，可用于相同Response结构体复用
    - `ApiResponse schema` 增加 `$ref` 属性，用于指定由 `ApiDefinition` 定义的结构体
- 20200813
    - 增加Api版本, `ApiVersion`, 可以给路由增加版本前缀
    - 增加多服务支持, `ApiServer`, 可以按服务生成`swagger.json`
    - `ApiResponse shema` 支持字段简介
- 20200812
    - `body` 结构增加多级支持
    - `FormData` 增加 文件上传样例
    - 增加`swagger ui`命令行工具
