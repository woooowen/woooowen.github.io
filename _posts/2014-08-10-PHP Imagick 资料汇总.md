---
layout: post
title: Php Imagick常用知识
category: [PHP]
tags: [PHP,Imagick]
---
 
 根据公司现有情况,以及需求分析,这个需求使用imagick来做,涉及到的工作量就要相对少一点(例如不用在服务器上装组件,且公司不提倡使用第三方接口,而移动端没有相应得截图工具,他得截图只能截取一屏幕,一屏之外的无法截取.)

###需求

当用户分享某个用户到微博的时候,需要将该用户的以下信息合并成一整个图片分享出去

	1.用户封面图

	2.用户名

	3.用户描述(后剔除,因为样式不好看,且部分用户会用字符拼接一个表情,中文字体失效)

	4.粉丝数目红色显示,第一张twi图,和twi文字

最后这个需求正确得思路,应该是`html->pdf->imgick->pic`

htmltopdf 有第三方公司提供接口http://pdfcrowd.com/

嵌套html页面,然后通过接口或者在公司集群安装htmltopdf组件,然后根据html生成了pdf文件,再使用imgick转换并且优化成图片,还可以进行裁剪,压缩等处理.生成你要的图片. 下面主要使用imagick来完成这个需求.避免在集群安装htmltopdf.这样一个不常用的组件.

###解决思路

	1.由于没有图片得绝对地址,因此通过获取到的相对地址,构造出cdn地址.然后将需要用到的图片通过cdn地址下载到当前服务器下./var/www/html/www.xxx.com/Photo/top/ 类似该目录

	2.能够复用的图片尽量放在当前服务器上,减少网络开销,加快程序运行速度.

	3.图片处理

	4.生成大图,保存当前服务器

	5.将图片服务器zupload到FTS,然后返回相对地址

	6.将新生成的图片cdn地址通过微博接口分享出去.微博自动抓取图片显示

这边的主要展示图片处理部分

###常用方法

主要说说比较常用的方法以及遇到的坑,具体的数据获取就不说了.

####1.新建基础图片

{% highlight js %}

public function create($width,$height,$bgColor = 'white',$ext = 'jpg'){

	$image = new Model_Top_Imagick();

	$image->newimage($width,$height,new ImagickPixel($bgColor));

	$image->setimageformat($ext);

return $image;

}

{% endhighlight %}

####2.裁剪图片

{% highlight js %}

public function crop($x = 0,$y = 0,$width = null,$height = null){

	$myWidth = $this->getimagewidth();

	$myHeight = $this->getimageheight();

	//只有当图片宽度小于需要的时候才会拉伸图片.高度的话不会拉伸

	if($width > $myWidth)

	$this->thumbnailImage($width,'999',true);

	if($width == null)

		$width = $this->getImageWidth() - $x;

	if($height == null)

		$height = $this->getImageHeight() - $y;

	if($width <= 0 || $height <= 0)

		return;

	//$this->thumbnailImage($width,$height,true);

	$this->cropImage($width,$height,$x,$y);

}

{% endhighlight %}

需要注意的是,当原有图片比你需要的图片小,而你需要重新生成缩略图得时候,给高度设置999,然后在使用cropImage裁剪固定尺寸的方法比较好用. thumbnailImage($width,'999',true); 第三个参数为true表示保持比例进行放大缩小,false表示强制放大缩小,不保持比例.

####3.添加文字水印

{% highlight js %}
	$draw = new ImagickDraw();
	//设置字体
	if(isset($style['font_path'])) $draw->setFont($style['font_path']);

	if(isset($style['font_size'])) $draw->setFontSize($style['font_size']);

	if(isset($style['font_color'])) $draw->setFillColor($style['font_color']);

	if(isset($style['under_color'])) $draw->setTextUnderColor($style['under_color']);
{% endhighlight %}

当你定义文字水印的时候,imagick中需要指定字体,不然会出现乱码

{% highlight js %}
	'uName' => array(

	'font_size' => 36,

	'font_color' => '#333333',

	'font_path' => '/var/www/html/www.mogujie.com/public/fonts/fzlt_GBK.TTF'

	), //例如这样,指定字体路径
{% endhighlight %}

####4.添加图片

{% highlight js %}
public function addImg($obj,$x,$y){

	$this->setimagecolorspace($obj->getImageColorspace());

	$this->compositeImage($obj,$obj->getImageCompose(),$x,$y);

}
$imageObj['avatar'] = $obj->load($imgPath['avatarCircle']);

$imageObj['main']->addImg($imageObj['avatar'],260,$avatarHeight);
{% endhighlight %}

#####5.圆形图片生成

{% highlight js %}
$mask = $this->load('/var/www/html/www.mogujie.com/Photo/tmp/circle.png');

//外面的白色边框圆圈图片

//$border = $this->load('/var/www/html/www.mogujie.com/Photo/tmp/circle1.png');

$image = $this->load($avatarPath);

$image->setimagealphachannel(1);

$image->cropimage(130, 130, 0, 0);

$image->compositeImage($mask, Imagick::COMPOSITE_COPYOPACITY, 0, 0);

if($image->writeImage($outfile)){

	return $outfile;

}

return false;
{% endhighlight %}
这边是一个坑,Top的头像使用的jpg格式的,而在jpg格式中处理透明背景的时候,imagick会给它一个默认的颜色,默认颜色是黑色,因此会出现类似这样得结果.

2359446723avatar_circle

这个时候就算你先将jpg图片转换成png格式的,还是失败,需要将图片得alpha通道值设为1.$image->setimagealphachannel(1);

compositeImage,这个方法的用处主要是用来合并图片,第三个参数比较重要,具体的方法可以参见这个地址 http://us2.php.net/manual/zh/imagick.constants.php#imagick.constants.channel

imagick网上的资料比较少,具体的API可以看这个链接 http://xiaochaozi.blog.51cto.com/6469085/1302801

关于圆形图片那个,如果看不懂,可以参考这2篇文章

<a href="https://trovepromo-tf.trove-stg.com/0m1-sd/stackoverflow.com/questions/13795535/circularize-an-image-with-imagick">
https://trovepromo-tf.trove-stg.com/0m1-sd/stackoverflow.com/questions/13795535/circularize-an-image-with-imagick</a>

<a href="http://stackoverflow.com/questions/8699228/how-to-use-imagick-to-merge-and-mask-images">http://stackoverflow.com/questions/8699228/how-to-use-imagick-to-merge-and-mask-images</a>

注意:这2个文章的demo都是使用png格式的图片,如果你使用的图片是jpg得记得看我上面说的添加一条`setimagealphachannel(1);`方法,解决透明色显示默认黑色的问题.

 