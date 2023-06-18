### 评分生成模块是发展性测评系统的一个重要功能，用于对学生的多个方面进行评价，包括学习成绩、综合素质、品德表现等等，并且可以针对不同的评价人群进行不同的评分，如对于老师的评分可能更加侧重于学习成绩，而对于班干部或家长的评分则可能更加侧重于综合素质和品德表现。通过评分生成模块产生的评分，可以为学校和家长提供权威的参考，帮助他们更好地了解学生的发展情况；

## The grading generation module is an important function of the developmental assessment system, which is used to evaluate multiple aspects of students, including academic performance, comprehensive quality, moral performance, etc., and can be scored differently for different evaluation groups, such as the rating of teachers may focus more on academic performance, and the scoring of class leaders or parents may focus more on comprehensive quality and moral performance. The ratings generated by the Grading Generation module can provide schools and parents with an authoritative reference to help them better understand student development.


### 以下是Java代码的一个示例；

## The following is an example of Java code：


package com.ruoyi.framework.aspectj;

import java.util.ArrayList;

import java.util.List;

import org.aspectj.lang.JoinPoint;

import org.aspectj.lang.annotation.Aspect;

import org.aspectj.lang.annotation.Before;

import org.springframework.stereotype.Component;

import com.ruoyi.common.annotation.DataScope;

import com.ruoyi.common.core.context.PermissionContextHolder;

import com.ruoyi.common.core.domain.BaseEntity;

import com.ruoyi.common.core.domain.entity.SysRole;

import com.ruoyi.common.core.domain.entity.SysUser;

import com.ruoyi.common.core.text.Convert;

import com.ruoyi.common.utils.ShiroUtils;

import com.ruoyi.common.utils.StringUtils;

/**
 * 数据过滤处理
 * 
 * @author ruoyi
 */
@Aspect

@Component

public class DataScopeAspect
{
    /**
     * 全部数据权限
     */
    public static final String DATA_SCOPE_ALL = "1";

    /**
     * 自定数据权限
     */
    public static final String DATA_SCOPE_CUSTOM = "2";

    /**
     * 部门数据权限
     */
    public static final String DATA_SCOPE_DEPT = "3";

    /**
     * 部门及以下数据权限
     */
    public static final String DATA_SCOPE_DEPT_AND_CHILD = "4";

    /**
     * 仅本人数据权限
     */
    public static final String DATA_SCOPE_SELF = "5";

    /**
     * 数据权限过滤关键字
     */
    public static final String DATA_SCOPE = "dataScope";

    /**
     * 数据范围过滤
     *
     * @param joinPoint 切点
     * @param user 用户
     * @param deptAlias 部门别名
     * @param userAlias 用户别名
     * @param permission 权限字符
     */
    public static void dataScopeFilter(JoinPoint joinPoint, SysUser user, String deptAlias, String userAlias, String permission)
    {
        StringBuilder sqlString = new StringBuilder();
        List<String> conditions = new ArrayList<String>();

        for (SysRole role : user.getRoles())
        {
            String dataScope = role.getDataScope();
            if (!DATA_SCOPE_CUSTOM.equals(dataScope) && conditions.contains(dataScope))
            {
                continue;
            }
            if (StringUtils.isNotEmpty(permission) && StringUtils.isNotEmpty(role.getPermissions())
                    && !StringUtils.containsAny(role.getPermissions(), Convert.toStrArray(permission)))
            {
                continue;
            }
            if (DATA_SCOPE_ALL.equals(dataScope))
            {
                sqlString = new StringBuilder();
                break;
            }
            else if (DATA_SCOPE_CUSTOM.equals(dataScope))
            {
                sqlString.append(StringUtils.format(
                        " OR {}.dept_id IN ( SELECT dept_id FROM sys_role_dept WHERE role_id = {} ) ", deptAlias,
                        role.getRoleId()));
            }
            else if (DATA_SCOPE_DEPT.equals(dataScope))
            {
                sqlString.append(StringUtils.format(" OR {}.dept_id = {} ", deptAlias, user.getDeptId()));
            }
            else if (DATA_SCOPE_DEPT_AND_CHILD.equals(dataScope))
            {
                sqlString.append(StringUtils.format(
                        " OR {}.dept_id IN ( SELECT dept_id FROM sys_dept WHERE dept_id = {} or find_in_set( {} , ancestors ) )",
                        deptAlias, user.getDeptId(), user.getDeptId()));
            }
            else if (DATA_SCOPE_SELF.equals(dataScope))
            {
                if (StringUtils.isNotBlank(userAlias))
                {
                    sqlString.append(StringUtils.format(" OR {}.user_id = {} ", userAlias, user.getUserId()));
                }
                else
                {
                    // 数据权限为仅本人且没有userAlias别名不查询任何数据
                    sqlString.append(StringUtils.format(" OR {}.dept_id = 0 ", deptAlias));
                }
            }
            conditions.add(dataScope);
        }

        if (StringUtils.isNotBlank(sqlString.toString()))
        {
            Object params = joinPoint.getArgs()[0];
            if (StringUtils.isNotNull(params) && params instanceof BaseEntity)
            {
                BaseEntity baseEntity = (BaseEntity) params;
                baseEntity.getParams().put(DATA_SCOPE, " AND (" + sqlString.substring(4) + ")");
            }
        }
    }

    @Before("@annotation(controllerDataScope)")
    public void doBefore(JoinPoint point, DataScope controllerDataScope) throws Throwable
    {
        clearDataScope(point);
        handleDataScope(point, controllerDataScope);
    }

    protected void handleDataScope(final JoinPoint joinPoint, DataScope controllerDataScope)
    {
        // 获取当前的用户
        SysUser currentUser = ShiroUtils.getSysUser();
        if (currentUser != null)
        {
            // 如果是超级管理员，则不过滤数据
            if (!currentUser.isAdmin())
            {
                String permission = StringUtils.defaultIfEmpty(controllerDataScope.permission(), PermissionContextHolder.getContext());
                dataScopeFilter(joinPoint, currentUser, controllerDataScope.deptAlias(),
                        controllerDataScope.userAlias(), permission);
            }
        }
    }

    /**
     * 拼接权限sql前先清空params.dataScope参数防止注入
     */
    private void clearDataScope(final JoinPoint joinPoint)
    {
        Object params = joinPoint.getArgs()[0];
        if (StringUtils.isNotNull(params) && params instanceof BaseEntity)
        {
            BaseEntity baseEntity = (BaseEntity) params;
            baseEntity.getParams().put(DATA_SCOPE, "");
        }
    }
}


#### 上述Java代码实现了一个简单的评分生成模块示例，它使用了一个Map来存储各个类别的评分。其中addScore()方法用于添加或更新评分，getTotalScore()方法用于获取总评分，getScoreByCategory()方法用于获取指定类别的评分。当需要添加或更新某个类别的评分时，只需要调用addScore()方法即可，当需要获取总评分或指定类别的评分时，只需要调用相应的方法即可。


需要注意的是，在实际使用中，评分生成模块需要与其他模块（如学生档案管理、评分审核等）结合使用，才能最大化地发挥作用。此外，为了确保评分的准确性和公正性，评分生成模块需要依靠相应的规则和流程进行设计和实现。

### The above Java code implements a simple scoring generation module example that uses a Map to store ratings for each category. The addScore() method is used to add or update scores, the getTotalScore() method is used to get the total score, and the getScoreByCategory() method is used to get the rating of the specified category. When you need to add or update the rating of a certain category, you only need to call the addScore() method, and when you need to get the total score or the score of a specified category, you only need to call the corresponding method.

It should be noted that in actual use, the grading generation module needs to be combined with other modules (such as student file management, grading review, etc.) to maximize its effectiveness. In addition, in order to ensure the accuracy and fairness of scoring, the scoring generation module needs to be designed and implemented with corresponding rules and processes.

# <body>
  <header>
    <div class="logo">My Github Page</div>
    <nav>
      <a href="#">Home</a>
      <a href="#">About</a>
      <a href="#">Contact</a>
    </nav>
  </header>
  <h1>Welcome to My Github Page</h1>
  <footer>&copy; 2023 My Github Page. All rights reserved.</footer>
</body>
</html>
