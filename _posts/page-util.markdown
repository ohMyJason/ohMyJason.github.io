

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
