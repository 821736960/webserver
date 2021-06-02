# process_read
函数的返回值是对请求的文件分析后的结果，一部分是语法错误导致的BAD_REQUEST，一部分是do_request的返回结果.
do_request函数将网站根目录和url文件拼接，然后通过stat判断该文件属性。

项目中请求报文解析后的m_url有8种情况。

/ GET请求，跳转到judge.html，即欢迎访问页面  

/0 POST请求，跳转到register.html，即注册页面  

/1 POST请求，跳转到log.html，即登录页面  

/2CGISQL.cgi POST请求，进行登录校验  

验证成功跳转到welcome.html，即资源请求成功页面

验证失败跳转到logError.html，即登录失败页面

/3CGISQL.cgi POST请求，进行注册校验  

注册成功跳转到log.html，即登录页面

注册失败跳转到registerError.html，即注册失败页面

/5 POST请求，跳转到picture.html，即图片请求页面  

/6 POST请求，跳转到video.html，即视频请求页面  

/7 POST请求，跳转到fans.html，即关注页面  

```cpp
//网站根目录，文件夹内存放请求的资源和跳转的html文件
const char* doc_root="/home/qgy/github/ini_tinywebserver/root";

http_conn::HTTP_CODE http_conn::do_request()
{
    //将初始化的m_real_file赋值为网站根目录
    strcpy(m_real_file, doc_root);
    int len = strlen(doc_root);
    
    //printf("m_url:%s\n", m_url);
    //找到m_url中/的位置
    const char *p = strrchr(m_url, '/');

    //处理cgi
    if (cgi == 1 && (*(p + 1) == '2' || *(p + 1) == '3'))
    {

        //根据标志判断是登录检测还是注册检测
        char flag = m_url[1];

        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/");
        strcat(m_url_real, m_url + 2);
        strncpy(m_real_file + len, m_url_real, FILENAME_LEN - len - 1);
        free(m_url_real);

        //将用户名和密码提取出来
        //user=123&passwd=123
        char name[100], password[100];
        int i;
        for (i = 5; m_string[i] != '&'; ++i)
            name[i - 5] = m_string[i];
        name[i - 5] = '\0';

        int j = 0;
        for (i = i + 10; m_string[i] != '\0'; ++i, ++j)
            password[j] = m_string[i];
        password[j] = '\0';

        //同步线程登录校验
        if (*(p + 1) == '3')
        {
            //如果是注册，先检测数据库中是否有重名的
            //没有重名的，进行增加数据
            char *sql_insert = (char *)malloc(sizeof(char) * 200);
            strcpy(sql_insert, "INSERT INTO user(username, passwd) VALUES(");
            strcat(sql_insert, "'");
            strcat(sql_insert, name);
            strcat(sql_insert, "', '");
            strcat(sql_insert, password);
            strcat(sql_insert, "')");

            if (users.find(name) == users.end())
            {

                m_lock.lock();
                int res = mysql_query(mysql, sql_insert);
                users.insert(pair<string, string>(name, password));
                m_lock.unlock();

                if (!res)
                    strcpy(m_url, "/log.html");
                else
                    strcpy(m_url, "/registerError.html");
            }
            else
                strcpy(m_url, "/registerError.html");
        }
        //如果是登录，直接判断
        //若浏览器端输入的用户名和密码在表中可以查找到，返回1，否则返回0
        else if (*(p + 1) == '2')
        {
            if (users.find(name) != users.end() && users[name] == password)
                strcpy(m_url, "/welcome.html");
            else
                strcpy(m_url, "/logError.html");
        }
    }
    //如果请求资源为/0，表示跳转注册界面
    if (*(p + 1) == '0')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/register.html");
        //将网站目录和/register.html进行拼接，更新到m_real_file中
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    //如果请求资源为/1，表示跳转登录界面
    else if (*(p + 1) == '1')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        //将网站目录和/log.html进行拼接，更新到m_real_file中
        strcpy(m_url_real, "/log.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '5')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/picture.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '6')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/video.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else if (*(p + 1) == '7')
    {
        char *m_url_real = (char *)malloc(sizeof(char) * 200);
        strcpy(m_url_real, "/fans.html");
        strncpy(m_real_file + len, m_url_real, strlen(m_url_real));

        free(m_url_real);
    }
    else
        strncpy(m_real_file + len, m_url, FILENAME_LEN - len - 1);
        
   //通过stat获取请求资源文件信息，成功则将信息更新到m_file_stat结构体
    //失败返回NO_RESOURCE状态，表示资源不存在
    if (stat(m_real_file, &m_file_stat) < 0)
        return NO_RESOURCE;
    //判断文件的权限，是否可读，不可读则返回FORBIDDEN_REQUEST状态
    if (!(m_file_stat.st_mode & S_IROTH))
        return FORBIDDEN_REQUEST;
    //判断文件类型，如果是目录，则返回BAD_REQUEST，表示请求报文有误
    if (S_ISDIR(m_file_stat.st_mode))
        return BAD_REQUEST;
    //以只读方式获取文件描述符，通过mmap将该文件映射到内存中
    int fd = open(m_real_file, O_RDONLY);
    m_file_address = (char *)mmap(0, m_file_stat.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    //避免文件描述符的浪费和占用
    close(fd);
    //表示请求文件存在，且可以访问
    return FILE_REQUEST;
}
```
# process_write
根据do_request的返回状态，服务器子线程调用process_write向m_write_buf中写入响应报文。
HTTP响应由四个部分组成，分别是：状态行、消息报头、空行和响应正文。
```cpp
 1HTTP/1.1 200 OK  //状态行，由HTTP协议版本号， 状态码， 状态消息 三部分组成。（HTTP/1.1）表明HTTP版本为1.1版本，状态码为200，状态消息为OK。
 2Date: Fri, 22 May 2009 06:07:21 GMT //消息报头，用来说明客户端要使用的一些附加信息。
 3Content-Type: text/html; charset=UTF-8//第二行和第三行为消息报头，Date:生成响应的日期和时间；Content-Type:指定了MIME类型的HTML(text/html),编码类型是UTF-8。
 4空行//空行，消息报头后面的空行是必须的。
 5<html>//响应正文，服务器返回给客户端的文本信息。空行后面的html部分为响应正文。
 6      <head></head>
 7      <body>
 8            <!--body goes here-->
 9      </body>
10</html>
```
add_status_line函数，添加状态行：http/1.1 状态码 状态消息

add_headers函数添加消息报头，内部调用add_content_length和add_linger函数

content-length记录响应报文长度，用于浏览器端判断服务器是否发送完数据

connection记录连接状态，用于告诉浏览器端保持长连接

add_blank_line添加空行







