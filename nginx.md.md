
```nginx
# 当后端服务有代理前缀时, 比如后端的地址是localhost:8080/swagger-ui/index.html
# 但是暴露的是localhost:80/-/api/swagger-ui/index.html
server {
    listen       80;
    server_name  localhost;

    location /-/api/ {
        proxy_pass   http://127.0.0.1:8080/;
        proxy_set_header X-Forwarded-Prefix /-/api/;
    }
}

# 服务端需配置:
server.forward-headers-strategy: framework


# 拦截某个请求返回固定内容
location /test/api {
 default_type application/json;
 return 200 '{"code":0,"msg":null,"data":null}';
}
# 多行
location /test/api {
 default_type application/json;
 return 200 '{
     "code":0,
     "msg":null,
     "data":null
     }';
}

# 某些uri代理失败,使用try_files 
server {
    listen 7006;
    server_name localhost;
    
    location / {
       try_files $uri $uri/ /index.html;
       root html/app1;
    }
}

# 替换某个返回
location / {
        proxy_ssl_verify off; 
        proxy_ssl_server_name on;
        # 禁止zip
        proxy_set_header Accept-Encoding "";
        proxy_pass https://dev.moreinsight.ai;
        sub_filter 'https://dev.moreinsight.ai' '';
        sub_filter_once off;
        sub_filter_types text/css application/json application/javascript;
        proxy_hide_header Location;
        #add_header Location $new_location always;
}
```