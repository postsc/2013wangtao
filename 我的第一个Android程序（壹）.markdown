# 我的第一个Android程序（壹）

标签（空格分隔）： android

---

## WebService - 服务器端
第一次接到这个任务是实习所在公司的老板于八月份交代给我的，当时，我在和老板交流的时候说过在自学Android程序的开发，然后公司里正好有个后台网站可以做成手机客户端给我练手，于是乎，老板就给我布置下了这么个小小的任务（说句题外话，对于实习的公司、老板还有同事，我非常感谢，因为在这我能学到很多我所需要的，还有更多的是公司里的人都很棒！）。

拿到这个任务，就开始做前期的准备工作了。因为已经有一个成型的网站了，所以我就根据网站的逻辑模型照搬了一个手机端的逻辑结构。程序最主要的工作是访问远端的SqlServer数据库，然后在本地的手机屏幕上做呈现，基本的逻辑工作非常的清晰明了。但是在网络上做调查的时候，如果要用Android Apps访问数据库——甭管是MySql、SqlServer抑或是其他的RDBMS——都需要用到一种叫做WebService的服务，其用到的协议叫做“简单对象访问协议”（Simple Object Access Protocol， SOAP）。而我们所要做的是用这个服务而非去研究这个服务，所以对于WebService的实现细节如何，我们并不关心，最重要的是我们要知道如何在Android Apps里面使用这个服务。本以为，Android会提供相关的库，但可惜的是：并没有！所以，只好在网上找其他人已经实现好的（如果没有现成的………………那你只能自己写了………………）。所幸的是，有一个名为Ksoap2的Android WebService库，这就可以大大的减轻我们的程序开发压力了。

不过需要注意的是，上面我们找的Ksoap2仅仅是让我们能够在本地Apps里面调用WebService的服务，而如果我们真的使响应的服务运行起来，那我们还需要一个运行在一定环境下提供WebService服务的模块（或组件，不是很清楚该如何称呼这个“东西”，或者直接称为**WebService服务器**）。

因为上述后台网站使用的编程语言是PHP，所以，我也使用PHP语言来实现WebService的服务器端。好嘛，又是一通先验知识的查找，不过还好，PHP本身就提供了对于WebService的支持并且还提供了两个类来做完成相应的工作，它们分别是：`SoapServer`和`SoapClient`。`SoapServer`用于服务器；`SoapClient`用于PHP环境的客户端，在这里我们可以用来检测`SoapServer`提供的服务是否正常以及代码的逻辑是否正确。接下来就是WebService服务器端的代码编写了。

先来看看server.php的实现：
``` PHP
<?php
require_once("soapHandle.class.php");

try{
	$server = new SoapServer(null, array('uri'=>"http://192.168.0.100/server.php"));
	$server->setClass('soapHandle');
	$server->handle();
}
catch(SoapFault $f){
	$f->faulString;
}
?>
```
很简单的一个routine，第二行*require_once*语句类似于C语言的*include*，用来引入用于处理WebService请求的代码文件；第五行用于创建一个WebService Server，第二个参数指定位置为指定URL下的server.php文件；第六行设置真正的用于请求的代码时*soapHandle*类，其定义于*soapHandle.class.php*文件中；第七行启动这个WebService Server。代码片的其他部分皆是用于处理异常的。

然后是soapHandle.class.php文件，这个文件是最主要的，因为所有逻辑处理都是在这个文件里面实现的（当然了，名字是你自己起的，不必定势）：
``` PHP
<?php
class soapHandle{
    var $db = '';
	
	public function __construct(){
		$this->db = new PDO("sqlsrv:Server=192.168.0.2;Database=basename", "username", "password");
	}
	
	//管理员登陆
	public function login($userCode, $passwd, $ip){

		$stCode = -1;
		$hashStr = '';
		$userName = '';

		$cmd = "exec [dbo].[PR_S_Login] :userCode, :password, :ip, :stCode, :hashStr, userName";
		$st = $this->db->prepare($cmd);
		$st->bindParam(":userCode", $userCode, PDO::PARAM_STR);
		$st->bindParam(":password", $passwd, PDO::PARAM_STR);
		$st->bindParam(":ip", $ip, PDO::PARAM_STR);

		$st->bindParam(":stCode", $stCode, PDO::PARAM_INT | PDO::PARAM_INPUT_OUTPUT, PDO::SQLSRV_PARAM_OUT_DEFAULT_SIZE);
		$st->bindParam(":hashStr", $hashStr, PDO::PARAM_STR | PDO::PARAM_INPUT_OUTPUT, 32);
		$st->bindParam(":userName", $userName, PDO::PARAM_STR | PDO::PARAM_INPUT_OUTPUT, 20);
		
		$st->execute();
		$st->nextRowset();

		return $stCode.".".$hashStr;
	}
}
?>
```
类soapHandle就是前面server.php文件中setClass语句调用的逻辑处理代码，函数__construct是构造函数，其中实现了一个数据库连接的实例$db，*sqlsrv*表明连接到的数据是SqlServer；函数login就是我根据数据库存储过程所编写的系统登录函数，具体执行的存储过程就是代码的第16行所调用的"exec",其他的bindParam都是设置存储过程里面的参数。具体的存储过程的相关知识，此处不再赘述。

然后，我写了一个实验用的文件用来测试server.php的编写是否正确，内容如下：
``` PHP
<?php
try{
    $client = new SoapClient(null, array('location'=>"http://192.168.0.100/server.php",'uri'=>"http://192.168.0.100/server.php"));
	
	$stCode = 0;
	$userCode = "test";
	$passwd = md5("asdf");
	$ip = "192.168.0.102";
	
	$res = $client->login($userCode, $passwd, $ip);

	echo $res;
}
catch(SoapFault $f){
    $f->getMessage();
}
?>
```
如果成功调用会返回一个哈希值，如果失败会打印相应的信息。




