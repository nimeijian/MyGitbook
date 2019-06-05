在分析LCN的源码之前，我们先看下LCN的工作原理，以下是其工作原理图：（请参考\[https://www.txlcn.org/zh-cn/docs/principle/control.html\]\(https://www.txlcn.org/zh-cn/docs/principle/control.html\)）

!\[图片\]\(https://uploader.shimo.im/f/uh5iec2n2XorMc26.png!thumbnail\)

核心步骤

1、创建事务组

是指在事务发起方开始执行业务代码之前先调用TxManager创建事务组对象，然后拿到事务标示GroupId的过程。

2、加入事务组

添加事务组是指参与方在执行完业务方法以后，将该模块的事务信息通知给TxManager的操作。

3、通知事务组

是指在发起方执行完业务代码以后，将发起方执行结果状态通知给TxManager,TxManager将根据事务最终状态和事务组的信息来通知相应的参与模块提交或回滚事务，并返回结果给事务发起方。



本文按照这三步的时序对LCN的核心源码进行解读（为了突出核心流程，其它代码会忽略）。

LCN框架支持LCN/TCC/TXC三种模式，本文按照LCN、TCC和TXC的顺序一一分析。

LCN版本：5.0.2-RELEASE。

\# 一、LCN模式

LCN模式的工作原理为通过代理java.sql.Connection，在发起方和参与方commit/rollback本地事务时什么都不做，最后由协调器TM通知各参与方commit/rollback本地事务。（请参考\[https://www.txlcn.org/zh-cn/docs/principle/lcn.html\]\(https://www.txlcn.org/zh-cn/docs/principle/lcn.html\)）

\#\# 1、创建事务组

我们假设有两个系统：ServiceA和ServiceB，其中ServiceA为分布式事务的发起方，ServiceB为分布式事务的参与方，ServiceA调用ServiceB的服务。创建事务组由发起方ServiceA发起，这部分我们分析发起方创建事务组的逻辑。



入口TransactionAspect.runWithLcnTransaction\(\):

\`\`\`

@Around\("lcnTransactionPointcut\(\) && !txcTransactionPointcut\(\)" +

        "&& !tccTransactionPointcut\(\) && !txTransactionPointcut\(\)"\)

public Object runWithLcnTransaction\(ProceedingJoinPoint point\) throws Throwable {

    DTXInfo dtxInfo = DTXInfo.getFromCache\(point\);

    LcnTransaction lcnTransaction = dtxInfo.getBusinessMethod\(\).getAnnotation\(LcnTransaction.class\);

    dtxInfo.setTransactionType\(Transactions.LCN\);

    dtxInfo.setTransactionPropagation\(lcnTransaction.propagation\(\)\);

    dtxInfo.setTimeout\(lcnTransaction.timeout\(\)\);

    return dtxLogicWeaver.runTransaction\(dtxInfo, point::proceed\);

}

\`\`\`

