
# person_archive

```#mermaid
%%{
    init: {
        'theme': 'forest', 
        'themeVariables': { 
            'fontSize': '18px', 
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
    person_archive {
        id bigint PK
        sid varchar "第三方标志"
        xm varchar "name"
        sfz varchar "idcard"
        xb varchar "gender"
        nl int "age"
        mz varchar "nation"
        hjd varchar "census register"
        dh varchar "phone number"
        sjly varchar "data source"
        zdlx varchar "type"
        hyzk varchar "marital status"
        zzmm varchar "zhengzhi mianmao"
        zjxy varchar "faith"
        cp varchar "car plate"
        zy varchar "personal occupation"
        jg varchar "ji guan"
        czd varchar "permanent residence"
        xl varchar "educational background"
        deleted int
        created datetime
        creator bigint
        updated datetime
        updater bigint
    }    

    person_family_members {
        id bigint PK
        sid bigint "person source id"
        did bigint "person dest id"
        gx varchar "relation"        
    }
    
    person_archive ||--|{ person_family_members: ""

    person_tag {
        id bigint PK
        lx varchar "person type"
        jzrybh varchar 
        jzlb varchar
        jtzm varchar "specific crime"
        jzkssj varchar
        jzjssj varchar
        sftg varchar
    }

    person_archive ||--|{ person_tag: ""

```


# event

```#merimad
%%{
    init: {
        'theme': 'forest', 
        'themeVariables': { 
            'fontSize': '18px', 
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
    person_archive {
        id bigint PK
        sid varchar "第三方标志"
        xm varchar "name"
        sfz varchar "idcard"
        xb varchar "gender"
        nl int "age"
        mz varchar "nation"
        hjd varchar "census register"
        dh varchar "phone number"
        sjly varchar "data source"
        zdlx varchar "type"
        hyzk varchar "marital status"
        zzmm varchar "zhengzhi mianmao"
        zjxy varchar "faith"
        cp varchar "car plate"
        zy varchar "personal occupation"
        jg varchar "ji guan"
        czd varchar "permanent residence"
        xl varchar "educational background"
        deleted int
        created datetime
        creator bigint
        updated datetime
        updater bigint
    }    

    person_family_members {
        id bigint PK
        sid bigint "person source id"
        did bigint "person dest id"
        gx varchar "relation"        
    }
    
    person_archive ||--|{ person_family_members: ""

    person_tag {
        id bigint PK
        lx varchar "person type"
        jzrybh varchar 
        jzlb varchar
        jtzm varchar "specific crime"
        jzkssj varchar
        jzjssj varchar
        sftg varchar
    }

    person_archive ||--|{ person_tag: ""

```

314









---
# END