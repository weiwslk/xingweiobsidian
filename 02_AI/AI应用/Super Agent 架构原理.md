
![[Pasted image 20261108201700.png]]


DeepSeek 团队提出的一种新Agent架构
- 它解决什么问题？AI越能干，人为什么反而越忙？
- 任务由谁来协调？谁来分派任务、传递信息、跟进后续的变化？
- 工作怎样持续推进？需求变化或任务中断后，工作怎么继续接上？


Router + 多Worker
这套结构目前还是研究方向

把多窗口的调度工作，交给 Super Agent，人只跟Super Agent沟通
![[Pasted image 20261108202359.png]]


由AI结果调度工作，需要具备这四个特征
![[Pasted image 20261108202644.png]]

Super Agent 怎么用？先选择运行方式
![[Pasted image 20261108203416.png]]


![[Pasted image 20261108203656.png]]


![[Pasted image 20261108203729.png]]


进入实操：先按照DeepSeek Harness 桌面端
![[Pasted image 20261108204125.png]]


用自然语言，让Agent帮你完成本地部署
![[Pasted image 20261108204333.png]]

部署完成后，Super Agent 在哪里运行？
![[Pasted image 20261108205131.png]]




