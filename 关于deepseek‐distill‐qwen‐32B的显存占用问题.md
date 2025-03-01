关于今天上午会议上deepseek是跑在cpu还是gpu上的问题，我做了几个实验，保持indexer和reranker的device都为"cpu"不变的情况下，修改generator的device设置：

1- generator的device设置为"cpu"：

A770的显存基本不会被占用，利用率也是0%：
![image](https://github.com/user-attachments/assets/5605359e-05df-4f28-98d5-a2ac54e2e5f7)

进程列表里也看不到占用显存的进程：
![image](https://github.com/user-attachments/assets/e433498e-c3f2-45b7-a549-f5d3bd0edd46)

回答问题时cpu占用率会稳定在800%上下，内存占用率在31.5%上下：
![image](https://github.com/user-attachments/assets/b4fa92e2-cb4f-4ee4-9405-76fa26c906a9)

2- generator的device设置为"auto"：

A770的显存被占满，利用率偶尔出现一次65%左右的波动：
![image](https://github.com/user-attachments/assets/6cc059e6-ca76-42f2-b100-d4963ea69fc8)

进程列表里可以看到，edgecraftrag.server占用了20G上下的显存（123%）：
![image](https://github.com/user-attachments/assets/ae59e38b-4ba8-4f3a-b0ff-9096107ec92c)

回答问题时cpu占用率会稳定在70%上下，内存占用率在17.7%上下：
![image](https://github.com/user-attachments/assets/5c18a151-69a4-466d-a128-abb7f5bea4e3)

我大概估算了一下，每10亿float32类型参数（4字节）会占用10^9*4/1024/1024/1024 ~= 3.73G ~= 4G，依次可以推出：

fp16类型占用4G/2=2G，
int8类型占用2G/2=1G,
int4类型占用1G/2=0.5G,

依据这个可以推出，int4量化后的32B模型占用显存大约在：32*0.5=16(G)，且会略小于16G，因为每billion的显存占用为了方便计算进行了上取整（3.73G->4G），所以从理论上看，32B的模型量化到int4之后，应该可以跑在16G显存的显卡上。

和上面实验现象结合，我觉得模型确实是跑在了A770上。

对于低利用率、高显存占用的现象，我的理解是，量化到int4之后，模型跑在了16G显存的A770上，但是因为cpu效率不高，导致gpu只是偶尔被利用，不能处在长期高利用率的状态，换性能高一些的cpu可能可以解决这个问题。
