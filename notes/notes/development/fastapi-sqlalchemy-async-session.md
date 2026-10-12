# FastAPI 中管理 SQLAlchemy 异步会话

FastAPI 的路由函数可以是 `async def`，数据库也可以使用异步驱动。关键是让每个请求拿到自己的 `AsyncSession`，完成工作后关闭会话；不要让多个并发任务共用同一个会话。

## 会话的生命周期

可以把数据库引擎和会话工厂创建一次，再用依赖为每个请求提供会话：

```python
from collections.abc import AsyncIterator
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

engine = create_async_engine(DATABASE_URL, pool_pre_ping=True)
SessionFactory = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionFactory() as session:
        yield session
```

`async with` 会在请求结束后关闭会话，即使中途抛出异常也能清理资源。引擎和连接池通常由应用生命周期管理；进程关闭时再调用 `await engine.dispose()`。

## 让事务边界清楚

涉及多条写入时，用事务块表达“全部成功或全部回滚”：

```python
from sqlalchemy import select

async def rename_user(session: AsyncSession, user_id: int, new_name: str):
    async with session.begin():
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        user = result.scalar_one()
        user.name = new_name
```

事务块正常退出时提交，异常退出时回滚。简单查询可以直接 `await session.execute(...)`；读取单个结果时，再按预期选择 `scalar_one()`、`scalar_one_or_none()` 等结果方法。

## 常见边界

- `AsyncSession` 是有状态的事务对象，不是可以任意并发共享的连接池。一个请求或一个并发任务使用独立会话。
- `async` 不会让同步数据库驱动自动变成异步。连接 URL 必须使用兼容的异步驱动。
- 不要在数据库事务里等待很慢的外部 HTTP 请求；缩短事务可以减少锁持有时间。
- 依赖注入负责会话生命周期，业务函数负责事务边界，两者分工明确更容易测试。

## 延伸阅读

- [SQLAlchemy asyncio 文档](https://docs.sqlalchemy.org/en/21/orm/extensions/asyncio.html)
- [SQLAlchemy Session 与并发规则](https://docs.sqlalchemy.org/en/20/orm/session_basics.html)
- [FastAPI 数据库教程](https://fastapi.tiangolo.com/tutorial/sql-databases/)
