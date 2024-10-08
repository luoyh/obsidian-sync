


# 功能说明

向上输出一切地图相关信息, 比如经纬度反查地址, 经纬度查询道路等级, 道路查询, 区域行政区查询, 周边范围查询等.
向下对接第三方地图厂商, 可配置化, 支持多个厂商, 不同公司支持自定义资源查询.



```mermaid
sequenceDiagram
  autonumber
  调用方 ->> Map service: 查询地图信息
  critical 查询本地数据库
    Map service -->> Local DB: 查询本地是否存在数据
    option 本地数据存在
      Local DB ->> +Map service: 有结果
      Map service ->> -调用方: 返回结果
    option 本地数据不存在
      Local DB ->> +Map service: 没有结果
      Map service -->> -第三方: 根据配置查询第三方
    option 请求成功
      第三方 ->> +Map service: 返回结果
      Map service ->> +Local DB: 更新本地数据库
      Map service ->> -调用方: 返回结果
    option 第三方超时或异常
      第三方 ->> +Map service: 无数据
      Map service ->> -调用方: 无数据
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
        code string "编码,系统目前支持的第三方编码,如:baidu/amap/siwei"
        name string "名称"
        cfg string "配置信息,比如账户/appkey等,不同厂商不同"
        company_id bigint "公司id,0表示全局"
        prior int "优先级,越大越先使用"
        times int "默认使用次数限制,一般在全局情况下,默认给其他公司的次数,避免每个公司都初始化配置"
        enabled int "是否启用,0-否,1-是"
    }    

```

# 用户次数限制配置

当用户没有配置自定义的地图资源时, 平台给用户分配的资源使用次数

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
    map_company_limit {
      id bigint PK
      from_company bigint "哪个公司分配的,0表示平台,其它表示A公司能给B公司次数"
      company_id bigint "公司id"
      times int "次数,<=0表示无限制"
      used int "已使用次数"
      all int "总使用次数"
      type int "类型:0-总次数,1-每自然天,2-每自然月,3-每自然年"
      last_date int "最后请求日期,yyyyMMdd,用于重置次数"
    }
```


# 请求日志

记录每次请求的日志,方便后续做分析, 计费等.

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
    ods_map_log {
      id bigint PK
      vehicle_id bigint "车辆id"
      company_id bigint "公司id"
      client_id string "终端手机号"
      plate string "车牌"
      user_id string "用户id"
      code string "第三方code"
      cfg string "第三方配置,可加密存储"
      state int "状态:0-请求成功,1-请求失败"
      api string "请求api"
      dsc string "请求说明"
      qt datetime "请求时间"
      req string "请求参数"
      res string "返回结果"
    }
```

# 请求流程

```mermaid
flowchart LR
  A["请求开始"]
  A --> B{"查询自己配置的map_config"}
  B -->|无| C{查询分配的配置}
  C -->|无| E[结束]
  C -->|有| D{是否超数}
  D -->|无| F[查询第三方]
  D -->|超出限制| E
  F --> E
  B -->|有| F
```



# 数据表

存储地图服务的地址信息, 根据**经纬度做唯一约束**

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
    ods_map_data {
        lon int "经度,度乘以10的6次方,比如经度为:106.3412,存的结果是:106341200"
        lat int "纬度,和经度一样"
        formatted_address_poi string "结构化地址"
        formatted_address string "结构化地址（不包含POI信息）"
        edz string "所属开发区"
        pois string "周边poi数组"
        roads string "道路信息,包含道路等级"
        poi_regions string "查询点的poi信息"
        sematic_description string "当前位置结合POI的语义化结果描述"
        district string "行政区编码"
        district_text string "行政区划"
    }    
    
```

