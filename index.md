<?php 
namespace Admin\Controller;
use Think\Controller;
class CommonController extends Controller{
	public function __construct(){
		//先调用父类构造方法
		parent::__construct();
		//登陆判断
		if (!session('?manager_info')) {
			//没有登陆  跳转到登陆页
			$this -> redirect('Admin/Login/login');
		}
	}
}
?>




<?php 
namespace Admin\Controller;
use Think\Controller;
class GoodsController extends Controller{
	//商品列表页
	public function index(){
		//实例化Goods模型
		$model = D('Goods');
		$total = $model -> count();
		//实例化分页类
		$pagesize = 3;
		$page = new \Think\page($total,$pagesize);
		//分页栏内容定制
		$page -> setConfig("prev","上一页");
		$page -> setConfig("next","下一页");
		$page -> setConfig("first","首页");
		$page -> setConfig("last","末页");
		$page -> setConfig("theme","%HEADER% %FIRST% %UP_PAGE% %LINK_PAGE% %DOWM_PAGE% %END%");
		$page -> rollPage = 3;
		$page -> lastSuffix = false;
		//调用show方法   获取分页栏html代码
		$page_html = $page -> show();
		$this -> assign('page_html',$page_html);
		//当前页数据查询
		$data = $model -> limit($page->firstRow,$page->listRows) -> select();
		//变量赋值
		$this -> assign('data',$data);
		//调用模板
		$this -> display();
	}
	//商品新增页
	public function add(){
		//一个方法  处理两个业务逻辑    展示页面   表单提交
		if (IS_POST) {
			//post请求   表单提交
			$data = $_POST;
			$data['goods_introduce'] = I('post.introduce','','remove_xss');
			// $data['name'] = htmlspecialchars($data['name']);
			$data = I("post.");
			$data['goods_introduce'] = I('post.introduce','','trim');
			//将数据添加到数据表
			//实例化模型
			$model = D("Goods");
			//文件上传
			$uoload_res = $model -> upload_logo($data);
			if (!$uoload_res) {
				//文件上传失败
				$error = $model -> getError();
				$this -> error($error);
			}
			//获取旧照片路径  用于后续从磁盘删除旧照片
			$goods = $model -> where(['id' => $data['id']]) -> find();

			$create = $model -> create($data);
			//使用create方法自动创建数据集（包含数据验证，字段映射，自动验证等功能）
			// $create = $model -> create();
			if (!$create) {
				//创建数据失败 报错
				$error = $model -> getError() ?: "数据错误";
				$this -> error($error);
			}
			//调用模型中的add方法进行添加  传递一堆数组参数
			$res = $model -> add($data);
			//dump($res)
			if ($res) {
				//添加成功  $res 是商品表的主键id值
				//商品相册图片的上传
				$model -> upload_pics($res);
				//提示并跳转
				$this -> success('添加成功' , U('Admin/Goods/index'));
			}else{
				$this -> success('添加失败');
			}
		}else{
			//get请求  展示页面
			//调用模板
			$this -> display();
		}
	}
	public function edit(){
		//一个方法两个逻辑
		if (IS_POST) {
			//表单提交
			$data = I('post.');
			//对商品描述字段单独处理，防范xss攻击
			$data['goods_introduce'] = I('post.introduce','','remove_xss');
			//将数据保存在数据表
			//实例化模型
			$model = D('Goods');
			//文件上传
			if($_FILES['logo']['error'] == 0){
				//如果上传了新图片则进行处理
				$upload_res = $model -> upload_logo($data);
				if(!$upload_res){
					//上传过程中出错
					$error = $model -> getError();
					$this -> error($error);
				}
				//获取旧图片路径  用于后续从磁盘删除旧照片
				$goods = $model -> where(['id' => $data['id']]) -> find();
			}
			$res = $model -> save();
			if ($res !== false) {
				//修改成功
				//如果上传了新照片  从磁盘删除就logo图片
				if ($_FILES['logo']['error'] == 0){
					unlink(ROOT_PATH . $goods['goods_big_img']);
					unlink(ROOT_PATH . $goods['goods_small_img']);
				}
				//继续上传相册图片
				$model -> upload_pics($data['id']);
				
				$this -> sunccess('修改成功', U('Admin/Goods/index'));
			}else{
				//修改失败
				$this -> error('修改失败');
			}
		}else{
			//接受id参数，  查询原始数据
			$id = I('get.id');
			//查询数据
			$goods = D('Goods') -> where(['id' => $id]) -> find();
			//变量赋值
			$this -> assign('goods',$goods);

			//查询商品相册数据
			$goodspics = M('Goodspics') -> where(['goods_id' -> $id]) -> select();
			$this -> assign('goodspics', $goodspics);
			
			//调用模板
			$this -> display();
		}
	}
	//商品删除
	public function del(){
		//接收id参数
		$id = I('get.id');
		//从商品表删除记录
		$model = D('Goods');
		$res = $model -> where(['id' => $id]) -> delect();
		if ($res !== false) {
			$this -> success('删除成功',U('Admin/Goods/index'));
		}else{
			//删除失败
			$this -> error('删除失败');
		}
	}
	public function upload_pics($goods_id){
		//判断是否有文件需要被上传
		if (min( $_FILES['goods_pics']['error']) != 0){
			//所有文件都有错误
			return false;
		}
		//实例化Upload类
		//实例化文件上传类
		$config = array(
			'maxSize' => 10 * 1024 * 1024,//上传问价大小限制
			'exts' => array('jpg','png','gif','jpeg'),//允许上传的文件后缀
			'rootPath' => ROOT_PATH . UPLOAD_PATH,//保存根路径
		);
		$upload = new \Think\upload($config);
		//调用upload方法进行多文件上传
		$files = array($_FILES['goods_pics']);
		$res = $upload -> upload($files);
		if (!$res) {
			return false;
		}
		//上传成功 ，$res是一个二维数组
		foreach ($res as $key => $v) {
			//每一张图片，拼接原图片的路径，生成三张缩略图
			//然后向相册表添加一条记录
			$origin_pics = UPLOAD_PATH . $v['savepath'] . $v['savename'];

			//生成三张缩略图
			//实例化Image类
			$image = new \Think\Image();
			$image -> open(ROOT_PATH . $origin_pics);
			//生成800*800 缩略图
			$image -> thumb(800, 800);
			$pics_big = UPLOAD_PATH . $v['savepath'] . 'thumb800_' . $v['savename'];
			$image -> save(ROOT_PATH . $pics_big);
			//生成350*350 缩略图
			$image -> thumb(350,350);
			$pics_big = UPLOAD_PATH . $v['savepath'] . 'thumb350_' . $v['savename'];
			$image -> save(ROOT_PATH . $pics_mid);
			//生成50*50缩略图
			$image -> thumb(50, 50);
			$pics_sma = UPLOAD_PATH . $v['savepath'] . 'thumb50_' . $v['savename'];
			$image -> save(ROOT_PATH . $pics_sma);

			//添加一条数据到相册表
			$row = array(
				'goods_id' => $goods_id,
				'pics_origin' => $origin_pics,
				'pics_big' => $pics_big,
				'pics_mid' => $pics_mid,
				'pics_sma' => $pics_sma,
			);
			$goodspics[] = $row;
		}
		D('Goodspics') -> addAll($goodspics);
		return true;
	}
}
?>





<?php 
namespace Admin\Controller;
use Think\Controller;
class IndexController extends Controller{
	//后台首页
	public function index(){
		//图标需要的数据
		$data = [6,10,15,-5];
		$data = json_encode($data);
		$this -> assign('data',$data);
		//调用模板
		$this -> display();
	}
}
?>





<?php 
namespace Admin\Controller;
use Think\Controller;
class LoginController extends Controller{
	//后台登陆页
	public function login(){
		if (IS_POST) {
			//表单提交
			$username = I('post.username');
			$password = I('post.password');
			$code = I('post.code');
			//验证码校验
			//实例化Verify类，调用check方法
			$verify = new \Think\Verify();
			$check = $verify ->check($code);
			if (!$check) {
				$this -> error('验证码错误');
			}
			//数据检测
			if (!$username || !$password) {
				//用户名 密码不能为空
				$this -> error('用户名密码不能为空');
			}
			//实例化模型  根据用户名查询用户表
			$user = D('Manager') -> where(['username' => $username]) -> find();
			if ($user && $user['password'] == encrypt_password($password)){
				//登陆成功
				//设置登录页面
				session('manager_info',$user);
				//跳转后台首页
				$this -> success("登陆成功",U('Admin/Index/index'));
			}else{
				//用户名或者密码错误
				$this -> error('用户名或者密码错误');
			}
		}else{
				//调用模板
				$this -> display();
		}
	}
	//完成后台登陆功能
	public function logout(){
		//销毁session
		session(null);
		//跳转到登陆页
		$this -> redirect("Admin/Login/login");
	}



	//验证码生成
	public function captcha(){
		//实例化验证码类Verify类
		$config = array(
			'length' => 4,//验证码位数
			'useCurve' => false,//是否画混交曲线
			'useNoise' => false,是否添加杂点
		);
		$verify = new \Think\Verify($config);
		//调用entry方法生成并输出验证码图片
		$verify -> entry();
	}
	public function ajaxlogin(){
		//表单提交
		$username = I('post.username');
		$password = I('post.password');
		$code = I('post.code');
		//验证码校验
		//实例化verify类  调用check方法
		$verify = new \Think\Verify();
		$check = $verify -> check($code);
		if (!$check) {
			$rerurn = array(
				'code' => 10001,
				'mag' => '验证码错误'
			);
			$this -> ajaxReturn($return);
		}
		//数据检测
		if (!$username || !$password){
			//用户名密码不能为空
			$return = array(
				'code' => 10002,
				'msg' => '用户名密码不能为空'
			);
			$this -> ajaxReturn($return);
		}
		//实例化模型   根据用户名查询用户表
		$user = D('Manager') -> where(['username' => $username]) -> find();
		if ($user && $user['password'] == encrypt_password($password) ){
			//登陆成功
			//设置登录标识
			session('manager_info',$user);
			//跳转后台首页
			$return = array(
				'code' => 10000,
				'msg' => '登陆成功'
			);
			$this -> ajaxReturn($return);
		}else{
			//用户名或者密码错误
			$return = array(
				'code' => 10003,
				'msg' => '用户名或者密码错误'
			);
			$this -> ajaxReturn($return);
		}
	}
}
?>





<?php 
namespace Admin\Controller;
use Think\Controller;
class ManagerController extends Controller{
	//管理员列表页
	public function index(){
		//实例化模型
		$model = D('Manager');
		$data = $model -> select();
		//变量赋值
		$this -> assign('data',$data);
		//调用模板
		$this -> display();
	}
}
?>





<?php 
namespace Admin\Controller;
use Think\Controller;
use Admin\Model\GoodsModel;
class TestController extends Controller{
	public function index(){
		// $model = new GoodsModel;
		// dump($model);
		// 
		// 快速实例化 M函数
		// $model = M('Goods')
		$model = M('Advice',null);
		dump($model);
	}
	public function chaxun(){
		//数据查询
		//实例化模型
		// $model = D("Goods");
		// //查询数据
		// //$data = $model -> find();
		// $data = $model -> find(2);
		// //$data = $model -> select();
		// dump($data);
		// 
		// 统计查询
		// $data = $model -> where("id > 4") -> count();
		// dump($data);
		// 
		
		//连表查询
		$advice_model = M('Advice', null);
		$data = $advice_model -> alias('a') -> field('a.*,u.username') -> join("left join tpshop_user as u on a.user_id") -> select();
		dump($data);

	}
	public function test_cookie(){
		//设置cookie
		cookie('name','zz');
		//读取
		$name = cookie('name');
		dump($name);

		//删除
		cookie('name',null);
		dump(cookie('name'));
		dump(cookie('team'));
	}
	//session函数
	public function test_session(){
		$user = ['username' => 'admin','nickname' => 'admin123'];
		session('user',$user);
		session('user.username','test');
		dump(session());
		//判断是否设置某个session
		dump(session('?user'));
	}
	//测试xss攻击的防范
	public function test_xss(){
		$name = 'test<script>alert("abc");</script>test';
		$name = remove_xss($name);

		dump($name);
	}
}
?>
