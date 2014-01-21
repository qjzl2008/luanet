#生存大挑战

手机网络游戏生存大挑战服务端的通用框架代码,包含gateserver和gameserver


大挑战是一款手机休闲网络游戏，游戏形式为开房间，每个房间内最大能容纳20名玩家游戏.

##服务器结构划分

服务器按功能分成3类:


1. redis服务器,负责数据的存储

2. 主逻辑服务器,战斗等游戏逻辑的处理

3. 网关服务器，负责用户的接入和账号验证


先期考虑一个服务器组能容纳5W人同时在线，需部署3台服务器，基本配置如下:

1. redis服务器, 4核心cpu,64GB内存,3T硬盘,高主频

2. 主逻辑服务器, 16核心cpu,64GB内存,1T硬盘,高主频

3. 网关服务器,8核心cpu,16GB内存,500GB硬盘,一般即可
 


服务器逻辑视图

												                --------------- 
																|redis service|
																---------------				
																	^
																	|
																	V						
			--------------------------------------------------------------------					
			|super service| battle service|battle service| .... |asyndb service|
			--------------------------------------------------------------------
				^                                                  
				|                                                   
				V			                                        
			------------------			                       
			|to super service|                                     
			------------------                                 
					
			---------------------------------------------------------------
			|agent service|agent service|agent service| ...|asyndb service|
			---------------------------------------------------------------



			client     client    client ... 




##网关用户id的设计

网关用户id用以唯一标识处于agent service中的玩家
	
	--------------------------------------
	|3位 agent service号|14用户ID号|15位递增值|
	--------------------------------------

如上图所示，用户在网关的唯一ID用一个32位整数标识:	

其中3位用于作为agent service号，取值0-7也就是一台物理机器最多可以加载8个agent service.

14位的用户编号,这个编号在agent service中唯一，也就是每个agent service最多容纳8192个玩家

15位递增值,在agent service内递增，每分配一个用户ID加1,用以防止从inter服务器中过来的消息被转发往一个已过期的用户.


##消息广播的处理

当inter服务器需要广播消息的时候,将需要接收消息的玩家的网关用户标识填入消息包尾部发往gateserver,由 to super service
接收,to inter service接收到包后直接推送到所有agent service的消息队列中，由agent serviece自己判断需要发往哪些用户.


##逻辑服务器的消息处理

玩家在逻辑服务器绑定一个唯一的消息分发器，这个消息分发器根据用户状态确定,如果用户在房间中则是其中一个battle service的消息分发器,否则是super service的消息分发器.消息分发器的绑定由super service处理,同一时刻用户数据只能由一个消息分发器修改，保证是线程安全的.绑定消息分发器后，来自网关服务的消息将会被路由到正确的消息处理器中.


##移动处理及同步

服务器采用2D网络作为地图，x,y坐标分别用16位无符号整数表示，则横竖能容纳一个65536X65536的网格,一个玩家站立的位置占4个格子,连个玩家的位置表示1米,则一个地图大小为16384X16384米.

玩家请求移动时由客户端发送一个目标点坐标，服务器根据地形信息计算路径，如果无法到达用户请求的目标点则到达一个可以到达的点上.服务器根据玩家的移动速度和路径计算出到达目标点的时间.将这个目标点和到达的时间点同步给视野内的客户端.只要服务端和客户端使用同样的寻路算法，即可保证路径一致.

为了处理用户新进入视野的问题，这个移动同步被作为一个玩家属性被同步出去，这样，新进入视野的时候这个属性也会被同步给能看到的客户端，防止了移动信息丢失.

在服务器中，这个移动处理分成两部分，第一部分是计算路径，第二部分是服务器的位置移动。
服务器的位置移动由一个固定间隔定时器处理，以固定间隔，根据玩家的移动速度，沿着路径移动，移动的同时刷AOI信息.

在移动的过程中，移动属性可能会变化，例如，在玩家进行一个长距离移动的过程中受到攻击,可能减缓其移动速度.则到达目标点的时间会被重新计算.


