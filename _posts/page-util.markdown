

```java

package com.hikvision.fj.util;

import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.hikvision.fj.entity.PageResult;
import com.hikvision.fj.constant.ResourceConst;
import com.hikvision.fj.constant.ServiceErrorCode;
import com.hikvision.fj.entity.PageQuery;
import com.hikvision.ga.common.BasePage;
import com.hikvision.ga.common.BaseResult;
import com.hikvision.ga.common.BusinessException;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.BeanUtils;
import org.springframework.core.task.TaskExecutor;

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.BiFunction;
import java.util.function.Function;

/**
 * <p>
 *
 * </p>
 *
 * @author : zhoubinghui
 * @version : 1.0
 * @date : 2022/9/22
 */
public class PageUtil {

    /**
     * 默认分页大小
     */
    private static final Integer DEFAULT_PAGE_SIZE = 1000;

    public static <T extends PageQuery, R> List<R> listAllByResult(Function<T, BaseResult<BasePage<R>>> function, T param) {
        int rn = 1;
        List<R> result = new ArrayList<>();
        param.setPageSize(Objects.nonNull(param.getPageSize()) ? param.getPageSize() : DEFAULT_PAGE_SIZE);
        for (int i = 0; i < rn; i++) {
            param.setPageNo(i + 1);
            BaseResult<BasePage<R>> applyResult = function.apply(param);
            if (!StringUtils.equals(ResourceConst.ResultCode.SUCCESS, applyResult.getCode())) {
                throw new BusinessException(ServiceErrorCode.ERR_INVOKE_INNER_INTERFACE.getCode(), "list all error," + applyResult.getMsg());
            }
            BasePage<R> apply = applyResult.getData();
            if (i == 0) {
                rn = (apply.getTotal().intValue() + param.getPageSize() - 1) / param.getPageSize();
            }
            result.addAll(apply.getList());
        }
        return result;
    }

    public static <T extends PageQuery, R> List<R> listAllByPageResult(Function<T, BaseResult<PageResult<R>>> function, T param) {
        int rn = 1;
        List<R> result = new ArrayList<>();
        param.setPageSize(Objects.nonNull(param.getPageSize()) ? param.getPageSize() : DEFAULT_PAGE_SIZE);
        for (int i = 0; i < rn; i++) {
            param.setPageNo(i + 1);
            BaseResult<PageResult<R>> applyResult = function.apply(param);
            if (!StringUtils.equals(ResourceConst.ResultCode.SUCCESS, applyResult.getCode())) {
                throw new BusinessException(ServiceErrorCode.ERR_INVOKE_INNER_INTERFACE.getCode(), "list all error," + applyResult.getMsg());
            }
            PageResult<R> apply = applyResult.getData();
            if (i == 0) {
                rn = (apply.getTotal() + param.getPageSize() - 1) / param.getPageSize();
            }
            result.addAll(apply.getRecords());
        }
        return result;
    }

    public static <T extends PageQuery, R> List<R> listAll(Function<T, BasePage<R>> function, T param) {
        int rn = 1;
        List<R> result = new ArrayList<>();
        param.setPageSize(Objects.nonNull(param.getPageSize()) ? param.getPageSize() : DEFAULT_PAGE_SIZE);
        for (int i = 0; i < rn; i++) {
            param.setPageNo(i + 1);
            BasePage<R> apply = function.apply(param);
            if (i == 0) {
                rn = (apply.getTotal().intValue() + param.getPageSize() - 1) / param.getPageSize();
            }
            result.addAll(apply.getList());
        }
        return result;
    }

    public static <T extends PageQuery, R> List<R> listAll(Function<T, BasePage<R>> function, T param, TaskExecutor taskExecutor) {
        int rn = 1;
        param.setPageSize(Objects.nonNull(param.getPageSize()) ? param.getPageSize() : DEFAULT_PAGE_SIZE);
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        param.setPageNo(1);
        BasePage<R> apply = function.apply(param);
        rn = (apply.getTotal().intValue() + param.getPageSize() - 1) / param.getPageSize();
        List<R> result = new CopyOnWriteArrayList<>(apply.getList());
        for (int i = 1; i < rn; i++) {
            int finalI = i;
            T t = null;
            try {
                t = (T) param.getClass().newInstance();
            } catch (InstantiationException | IllegalAccessException e) {
                throw new RuntimeException(e);
            }
            BeanUtils.copyProperties(param, t);
            T finalT = t;
            futures.add(CompletableFuture.runAsync(() -> {
                finalT.setPageNo(finalI + 1);
                result.addAll(function.apply(finalT).getList());
            }, taskExecutor));
        }
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        return result;
    }

    public static <T extends PageQuery, R> List<R> listAll(BiFunction<T, Class<R>, BasePage<R>> function, T param, Class<R> R) {
        int rn = 1;
        List<R> result = new ArrayList<>();
        param.setPageSize(Objects.nonNull(param.getPageSize()) ? param.getPageSize() : DEFAULT_PAGE_SIZE);
        for (int i = 0; i < rn; i++) {
            param.setPageNo(i + 1);
            BasePage<R> apply = function.apply(param, R);
            if (i == 0) {
                rn = (apply.getTotal().intValue() + param.getPageSize() - 1) / param.getPageSize();
            }
            result.addAll(apply.getList());
        }
        return result;
    }

    /**
     * 检测是否需要分页
     *
     * @param pageQuery 分页参数
     * @return
     */
    public static <T extends PageQuery> boolean checkRequiredPage(T pageQuery) {
        return Objects.nonNull(pageQuery.getPageNo()) && Objects.nonNull(pageQuery.getPageSize()) && !Objects.equals(pageQuery.getPageSize(), -1);
    }

    /**
     * 手动分页
     *
     * @param dataList 列表
     * @param pageNo   页数
     * @param pageSize 页大小
     * @param <T>      列表泛型
     * @return 分页结果
     */
    public static <T> BasePage<T> manualPage(List<T> dataList, int pageNo, int pageSize) {
        List<T> result = new ArrayList<>();
        int fromIndex = (pageNo - 1) * pageSize;
        if (fromIndex >= dataList.size()) {
            return new BasePage<>((long) dataList.size(), result);
        }
        return new BasePage<>((long) dataList.size(), dataList.subList(fromIndex, Math.min(fromIndex + pageSize, dataList.size())));
    }

    public static <T> BasePage<T> pageToBasePage(Page<T> page) {
        List<T> list = Optional.ofNullable(page.getRecords()).orElse(new ArrayList<>());
        return new BasePage<>(Objects.equals(page.getTotal(), 0) ? list.size() : page.getTotal(), list);
    }
}

```


```java
package com.hikvision.fj.util;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.TypeReference;
import com.hikvision.fj.config.SystemConfig;
import com.hikvision.ga.common.BaseResult;
import com.hikvision.ga.common.BusinessException;
import com.hikvision.starfish.crypto.identityauth.IdentityAuthenticator;
import log.HikLog;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

import javax.annotation.Resource;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * @author liujiawei16
 * @date 2023/10/23 17:14
 */
@Component
@Slf4j
public class RestUtil {
    //自定义RestTemplate
    @Resource(name = "nativeRestTemplate")
    RestTemplate httpsRestTemplate;

    //海星寻址RestTemplate,也可使用@autowired/@Resource获取海星@primary的restTemplate
    @Resource
    RestTemplate restTemplate;

    @Resource
    IdentityAuthenticator identityAuthenticator;

    @Resource
    SystemConfig systemConfig;

    public static final Integer LOG_SIZE = 200;

    public static final String SUCCESS = "0";

    public static Boolean isSuccess(BaseResult br) {
        return br != null && SUCCESS.equals(br.getCode());
    }

    /**
     * 通用restTemplate请求,兼容海康寻址和外部请求
     *
     * @param param        body实体参数（String/实体类/jsonObject）
     * @param url          请求url,分为普通url和海康寻址url
     * @param header       请求头,可自定义参数
     * @param t            返回类型引用<实体类>
     * @param type         请求方法
     * @param isHikRequest 是否是海康请求
     * @param <T>          返回实体类型
     * @param <P>          请求实体类型
     * @return 实体类
     */
    public <T, P> T request(String url, Map<String, String> header, P param, HttpMethod type, TypeReference<T> t, boolean isHikRequest) {
        HttpHeaders httpHeaders = new HttpHeaders();
        String resBody = null;
        Exception ex = null;
        try {
            if (header == null) {
                header = new HashMap<>();
            }
            header.put("Content-Type", "application/json;charset=UTF-8");
            for (Map.Entry<String, String> entry : header.entrySet()) {
                httpHeaders.add(entry.getKey(), entry.getValue());
            }

            // 如果是GET请求，将参数添加到URL中
            if (type == HttpMethod.GET && param != null) {
                UriComponentsBuilder builder = UriComponentsBuilder.fromHttpUrl(url);
                Map<String, Object> paramMap = null;
                if (param instanceof Map) {
                    paramMap = (Map<String, Object>) param;
                } else {
                    paramMap = convertObjectToMap(param);
                }

                for (Map.Entry<String, Object> entry : paramMap.entrySet()) {
                    builder.queryParam(entry.getKey(), entry.getValue());
                }
                url = builder.toUriString();
                param = null; // GET请求不需要请求体
            }

            HttpEntity<Object> hashMapHttpEntity = new HttpEntity<>(param, httpHeaders);

            try {
                resBody = (isHikRequest ? restTemplate : httpsRestTemplate)
                        .exchange(url, type, hashMapHttpEntity, String.class).getBody();
            } catch (Exception e) {
                log.error("调用过程异常, url:{}, 异常: ", url, e);
                ex = new RuntimeException("调用过程异常", e);

            }
            if (Objects.isNull(resBody)) {
                log.error("调用过程异常, url:{} ", url);
                ex = new RuntimeException(HikLog.toLogWithParam("{}返回为null", url));
                return null;
            } else {
                return JSONObject.parseObject(resBody, t);
            }

        } catch (Exception exg) {
            ex = new RuntimeException("过程处理异常", exg);
            return null;
        } finally {
            String logResBody = null;
            if (systemConfig.getLogReductionMode()) {
                logResBody = Optional.ofNullable(resBody).map((s) -> s.substring(0, Math.min(s.length(), LOG_SIZE)) + ".....").orElse("null");
            }
            log.info("\n=================start=================\n请求url:{}\n请求body:{}\n请求头部:{}\n返回参数:{}\n调用方式:{}\n异常:{}\n==================end================",
                    url, JSONObject.toJSONString(param), JSONObject.toJSONString(httpHeaders), logResBody, type, JSON.toJSONString(ex));
        }
    }


    public <T, P> T requestWithParam(String url, Map<String, String> header, P param, HttpMethod type, TypeReference<T> t, boolean isHikRequest, String title) {
        HttpHeaders httpHeaders = new HttpHeaders();
        String resBody = null;
        Exception ex = null;
        try {
            if (header == null) {
                header = new HashMap<>();
            }
            header.put("Content-Type", "application/json;charset=UTF-8");
            for (Map.Entry<String, String> entry : header.entrySet()) {
                httpHeaders.add(entry.getKey(), entry.getValue());
            }

            // 如果是GET请求，将参数添加到URL中
            if (type == HttpMethod.GET && param != null) {
                UriComponentsBuilder builder = UriComponentsBuilder.fromHttpUrl(url);
                Map<String, Object> paramMap = null;
                if (param instanceof Map) {
                    paramMap = (Map<String, Object>) param;
                } else {
                    paramMap = convertObjectToMap(param);
                }

                for (Map.Entry<String, Object> entry : paramMap.entrySet()) {
                    builder.queryParam(entry.getKey(), entry.getValue());
                }
                url = builder.toUriString();
                param = null; // GET请求不需要请求体
            }

            HttpEntity<Object> hashMapHttpEntity = new HttpEntity<>(param, httpHeaders);
            try {
                resBody = (isHikRequest ? restTemplate : httpsRestTemplate)
                        .exchange(url, type, hashMapHttpEntity, String.class).getBody();
            } catch (Exception e) {
                ex = new RuntimeException("调用过程异常", e);
            }
            if (Objects.isNull(resBody)) {
                ex = new RuntimeException(HikLog.toLogWithParam("{}返回为null", url));
                return null;
            } else {
                return JSONObject.parseObject(resBody, t);
            }

        } catch (Exception exg) {
            ex = new RuntimeException("过程处理异常", exg);
            return null;
        } finally {
            String logResBody = null;
            if (systemConfig.getLogReductionMode()) {
                logResBody = Optional.ofNullable(resBody).map((s) -> s.substring(0, Math.min(s.length(), LOG_SIZE)) + ".....").orElse("null");
            }
            log.info("\n=================start=================\n接口名称:{}\n请求url:{}\n请求body:{}\n请求头部:{}\n返回参数:{}\n调用方式:{}\n异常:{}\n==================end================",
                    title, url, JSONObject.toJSONString(param), JSONObject.toJSONString(httpHeaders), logResBody, type, JSON.toJSONString(ex));
        }

    }


    private Map<String, Object> convertObjectToMap(Object obj) {
        Map<String, Object> map = new HashMap<>();
        if (obj != null) {
            Class<?> currentClass = obj.getClass();
            while (currentClass != null) {
                Field[] fields = currentClass.getDeclaredFields();
                for (Field field : fields) {
                    try {
                        field.setAccessible(true);
                        Object value = field.get(obj);
                        if (value != null) {
                            map.put(field.getName(), value);
                        }
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
                currentClass = currentClass.getSuperclass();
            }
        }
        return map;
    }

    public String token() {
        return identityAuthenticator.getIdentityToken();
    }
}


```
```java
package com.hikvision.fj.util;

import org.apache.commons.lang3.ArrayUtils;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

import java.util.Map;


/**
 * 获取Spring的ApplicationContext对象工具，可以用静态方法的方式获取spring容器中的bean
 * @author https://blog.csdn.net/chen_2890
 * @date 2019/6/26 16:20
 */
@Component
public class SpringContextUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    private static boolean isDev;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextUtil.applicationContext = applicationContext;
        isDev = ArrayUtils.contains(getActiveProfile(), "dev");
    }

    /**
     * 获取applicationContext
     */
    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    /**
     * 通过name获取 Bean.
     */
    public static Object getBean(String name) {
        Object o = null;
        try {
            o = getApplicationContext().getBean(name);
        } catch (NoSuchBeanDefinitionException e) {
            // e.printStackTrace();
        }
        return o;
    }

    /**
     * 通过class获取Bean.
     */
    public static <T> T getBean(Class<T> clazz) {
        return getApplicationContext().getBean(clazz);
    }

    /**
     * 通过name,以及Clazz返回指定的Bean
     */
    public static <T> T getBean(String name, Class<T> clazz) {
        return getApplicationContext().getBean(name, clazz);
    }

    /**
     * 通过name获取 Bean.
     */
    public static <T> Map<String, T> getBeansOfType(Class<T> clazz) {
        return getApplicationContext().getBeansOfType(clazz);
    }

    /**
     * 获取配置文件配置项的值
     *
     * @param key 配置项key
     */
    public static String getEnvironmentProperty(String key) {
        return getApplicationContext().getEnvironment().getProperty(key);
    }

    /**
     * 获取spring.profiles.active
     */
    public static String[] getActiveProfile() {
        return getApplicationContext().getEnvironment().getActiveProfiles();
    }

    public static boolean isDev() {
        return isDev;
    }
}

```
```java
package com.hikvision.fj;

import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import io.swagger.annotations.ApiModelProperty;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;
import java.io.FileOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.*;


/**
 * 读取包内所有实体类根据注解生成sql
 *
 * @TableId 标识主键
 * @ApiModelProperty.value 标识注释
 * liujiasheng5
 * 2022年3月28日11:28:35
 * TODO: 2022/3/18 做成更通用的
 * TODO: 2022/3/29 自动生成mapper层，service代码
 */
public class SqlGenerator {
    //实体类所在的package在磁盘上的绝对路径
    public static final String packageName = "D:\\Working\\SourceSVN\\SSVNoffaccroom\\JKN20240705_955_1.0.0006\\offaccroom-web\\src\\main\\java\\com\\hikvision\\fj\\entity\\po";
    //项目中实体类的reference
    public static final String prefix = "com.hikvision.fj.entity.po";
    //生成sql的文件夹
//    public static final String filePath = "_pkg/windows/bin/tillegalpassdb";

    /**
     * 该类需要在实体类同module下
     * 输入实体类包绝对路径、包名引用前缀，·结尾
     *
     * @param args
     */
    public static void main(String[] args) {


        String className = "";

        StringBuffer sqls = new StringBuffer();
        //获取包下的所有类名称
        List<String> list = getAllClasses(packageName);
        for (String str : list) {
            className = prefix + "." + str.substring(0, str.lastIndexOf("."));
            String sql = generateSql(className, "");
            sqls.append(sql);
        }
        System.out.println(sqls.toString());

    }

    private static final Logger logger = LoggerFactory.getLogger(SqlGenerator.class);

    public static Map javaProperty2SqlColumnMap = new HashMap<>();

    static {
        javaProperty2SqlColumnMap.put("Integer", "int");
        javaProperty2SqlColumnMap.put("int", "int");
        javaProperty2SqlColumnMap.put("Short", "tinyint");
        javaProperty2SqlColumnMap.put("Long", "bigint");
        javaProperty2SqlColumnMap.put("long", "bigint");
        javaProperty2SqlColumnMap.put("BigDecimal", "decimal");
        javaProperty2SqlColumnMap.put("Double", "float4");
        javaProperty2SqlColumnMap.put("double", "float4");
        javaProperty2SqlColumnMap.put("Float", "float4");
        javaProperty2SqlColumnMap.put("boolean", "bool");
        javaProperty2SqlColumnMap.put("Boolean", "bool");
        javaProperty2SqlColumnMap.put("Timestamp", "timestamptz");
        javaProperty2SqlColumnMap.put("Date", "timestamptz");
        javaProperty2SqlColumnMap.put("String", "varchar");
    }


    /**
     * 根据实体类生成建表语句
     *
     * @param className 全类名
     * @author
     * @date
     */
    public static String generateSql(String className, String tableName) {
        try {
            Class clz = Class.forName(className);
            className = clz.getSimpleName();
            Field[] fields = clz.getDeclaredFields();
            //如果包含父类则拉取父类字段
            Class tempClz = clz;

            while ((tempClz.getSuperclass() != Object.class)) {
                tempClz = tempClz.getSuperclass();
                Field[] superFields = tempClz.getDeclaredFields();
                fields = fieldsConcat(superFields, fields);
            }


            //字段参数类型
            String param = "";
            //字段名
            String column = "";
            //组件字段名
            String keyField = "";

            StringBuilder sql = null;

            sql = new StringBuilder(50);

            if (tableName == null || ("").equals(tableName)) {
                TableName tableNameAnnotation = (TableName) clz.getAnnotation(TableName.class);
                tableName = tableNameAnnotation != null ? tableNameAnnotation.value() : "tb_" + tf2xhx(className);
            }

            sql.append("\n\n/*========= " + tableName + " ==========*/\n");
            sql.append("DROP TABLE IF EXISTS public." + tableName + "; \n");
            sql.append("CREATE TABLE public.").append(tableName).append(" ( \n");

            StringBuilder commentBuider = new StringBuilder();
            for (Field f : fields) {
                int modifiers = f.getModifiers();
                boolean isStatic = Modifier.isStatic(modifiers);

                if (isStatic) {
                    continue;
                }

                TableField tableField = f.getAnnotation(TableField.class);
                if (tableField != null) {
                    column = tableField.value();
                } else {
                    column = f.getName();
                }
                if (("serialVersionUID").equals(column)) {
                    continue;
                }
                column = tf2xhx(column);

                param = f.getType().getSimpleName();

                sql.append("    ").append(column).append(" ").append(javaProperty2SqlColumnMap.get(param));

                if (f.getAnnotation(TableId.class) != null) {
                    keyField = column;
                }

                sql.append(" null,\n");
                //读取注释
                if (f.getAnnotation(ApiModelProperty.class) != null) {
                    ApiModelProperty commonAno = f.getAnnotation(ApiModelProperty.class);
                    commentBuider.append("\nCOMMENT ON COLUMN public.").append(tableName).append(".").append(column).append(" IS '").append(commonAno.value()).append("';");
                }
            }
            sql.append("CONSTRAINT test_").append(tableName).append("_").append(keyField).append("_pk PRIMARY KEY (").append(keyField).append(") \n);");
            return sql.toString() + commentBuider.toString();

        } catch (ClassNotFoundException e) {
            logger.debug("该类未找到！");
            return null;
        }
    }

    /**
     * 字段数组合并
     *
     * @param a
     * @param b
     * @return
     */
    public static Field[] fieldsConcat(Field[] a, Field[] b) {
        List<Field> list = new ArrayList(Arrays.asList(a));
        list.addAll(Arrays.asList(b));
        return list.toArray(new Field[]{});
    }

    /**
     * 驼峰转下划线
     *
     * @param column
     * @return
     */
    public static String tf2xhx(String column) {
        //将大写字母转小写，并添加下划线
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < column.length(); i++) {
            char c = column.charAt(i);
            if (Character.isUpperCase(c) && i == 0) {
                sb.append(Character.toLowerCase(c));
            } else if (Character.isUpperCase(c) && i > 0) {
                sb.append("_" + Character.toLowerCase(c));
            } else {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    /**
     * 获取包下的所有类名称,获取的结果类似于 XXX.java
     *
     * @param packageName
     * @return
     * @author
     * @date
     */
    public static List getAllClasses(String packageName) {
        List classList = new ArrayList();
        String className = "";
        File f = new File(packageName);
        if (f.exists() && f.isDirectory()) {
            File[] files = f.listFiles();
            for (File file : files) {
                className = file.getName();
                classList.add(className);
            }
            return classList;
        } else {
            logger.debug("包路径未找到！");
            return null;
        }
    }
//
//    /**
//     * 将string 写入sql文件
//     *
//     * @param str
//     * @param path
//     * @author
//     * @date
//     */
//    public static void StringToSql(String str, String path) {
//        byte[] sourceByte = str.getBytes();
//        if (null != sourceByte) {
//            try {
//                //文件路径（路径+文件名）
//                File file = new File(path);
//                //文件不存在则创建文件，先创建目录
//                if (!file.exists()) {
//                    File dir = new File(file.getParent());
//                    dir.mkdirs();
//                    file.createNewFile();
//                }
//                //文件输出流用于将数据写入文件
//                FileOutputStream outStream = new FileOutputStream(file);
//                outStream.write(sourceByte);
//                outStream.flush();
//                outStream.close();
//                System.out.println("生成成功");
//            } catch (Exception e) {
//                e.printStackTrace();
//            }
//        }
//    }
}

```
