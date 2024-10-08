


# 功能说明

向上输出一切地图相关信息, 比如经纬度反查地址, 经纬度查询道路等级, 道路查询, 区域行政区查询, 周边范围查询等.
向下对接第三方地图厂商, 可配置化, 支持多个厂商, 不同公司支持自定义资源查询.



```mermaid
sequenceDiagram
  autonumber
  调用方 ->> Map: 查询地图信息
  critical 查询本地数据库
    Map -->> DB: 查询本地是否存在数据
  option 本地数据存在
    Map ->> 调用方: 返回结果
  option 本地数据不存在
    Map -->> 第三方: 根据配置查询第三方
  option 第三方超时或异常
    第三方 ->> Map: 无数据
    Map ->> 调用方: 无数据
  end
```



# 配置

主要用于配置第三方厂商资源信息, 默认有一个全局配置, 每个`客户`可使用自己的配置.

```mermaid
%%{
    init: {
        'theme': 'forest', 
        'themeVariables': { 
            'fontSize': '12px', 
            'fontFamily': 'Consolas',  
            'primaryColor': '#BB2528', 
            'primaryTextColor': '#fff', 
            'primaryBorderColor': '#7C0000', 
            'lineColor': '#F8B229', 
            'secondaryColor': '#006100', 
            'tertiaryColor': '#fff'
        }
    }
}%%

erDiagram
    map_config {
        id bigint PK
        code string "编码,系统目前支持的第三方编码:baidu/amap/siwei"
        name string "名称"
        cfg string "配置信息,比如账户/appkey等,不同厂商不同"
        company_id bigint "公司id,0表示全局"
        rank int "优先级,越大越先使用"
        failup int "0-否,1-是,失败后是否使用全局配置,只要有一个配置了,就会在所有失败后根据全局配置请求"
    }    
    
```


