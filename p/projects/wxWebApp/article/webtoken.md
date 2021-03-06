# 网页授权机制

[请参考微信公众平台开发者文档](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140842&token=&lang=zh_CN)

## 第一步 了解openid和unionid

1. openid可以唯一标识用户，每个用户针对每个公众号会产生一个安全的openid
2. unionid可以在多个公众号、移动应用做用户共通


## 第二步 了解授权机制

1. 需要在公众平台配置授权回调域名
2. 两种授权类型：静默授权获取用户openid、授权登录获取用户基本信息（实质都是跳转到授权链接进行授权然后又跳转回配置的回调页面链接，此时回调页面链接会带上code字段，可以通过code字段的值进行下一步业务，获取openid或者用户基本信息）
3. 静默授权机制：当跳转到静默授权链接进行授权后自动跳转回调页面，该行为用户无明显感知，只看到加载进度条发生了两次加载
4. 授权登录机制：当跳转到授权登录链接时需要用户决定是否同意登录，该行为如果发生在用户从公众号的会话或者菜单进入授权，则行为和静默授权一样


## 第三步 了解授权后如何通过code字段进行下一步业务

1. 静默授权：通过code获取openid
2. 授权登录：通过code获取openid和access_token，从而获取用户基本信息
3. 注意事项：
   - 以上业务需要在服务器端执行
   - 通过code换取的access_token，和基础支持获取的access_token不一样，后者用于调用JSSDK接口与公众号开发
   - 授权登录获取用户基本信息，和用户管理类接口中的“获取用户基本信息接口”不一样，后者是用户必须关注了公众号才会调用成功拿到数据


## 第四步 项目实战（附上代码）
> *后台需要做的配合：配合静默授权提供一个用户查重接口、配合授权登录提供一个新增用户接口*

1. 在服务器端执行获取openid或者用户基本信息的php脚本
   - 封装一个处理微信网页授权机制的类，命名为WebToken
   - 提供两个公共方法，getOpenid()，getUserInfo()

    ```php
    <?php
    require_once "DataManipulation.php";
    
    class WebToken extends DataManipulation {
        private $appId;
        private $appSecret;
        private $code;
    
        public function __construct($appId, $appSecret, $code) {
            $this->appId = $appId;
            $this->appSecret = $appSecret;
            $this->code = $code;
        }
    
        public function getOpenid() {
            
            $data = $this->getWebToken();
    
            return $data->openid;
        }
    
        public function getUserInfo() {
    
            $data = $this->getWebToken();
    
            $url = "https://api.weixin.qq.com/sns/userinfo?".
                   "access_token=$data->access_token&openid=$data->openid".
                   "&lang=zh_CN";
    
            $res = $this->httpGet($url);
    
            return $res;
        }
    
        private function getWebToken() {
    
            $url = "https://api.weixin.qq.com/sns/oauth2/access_token?".
                   "appid=$this->appId&secret=$this->appSecret&code=$this->code".
                   "&grant_type=authorization_code";
    
            $res = $this->httpGet($url);
    
            return json_decode($res);
        }
    }
    ?>
    ```

2. 响应前端网页授权的接口
   - 当请求state为scopeBase即静默授权时，返回openid
   - 当请求state为scopeUserinfo即授权登录时，以json数据格式返回用户基本信息
   
    ```php
    <?php
    require_once "class/WebToken.php";
    
    $code = isset($_GET["code"])?$_GET["code"]:null;
    $state = isset($_GET["state"])?$_GET["state"]:null;
    
    $heyQStr = trim(substr(file_get_contents("cache/heyQ.php"), 15));
    $heyQ = json_decode($heyQStr);
    
    $appId = $heyQ->appId;
    $appSecret = $heyQ->appSecret;
    
    $webToken = new WebToken($appId, $appSecret, $code);
    
    if($state === "scopeBase") {
        echo $webToken->getOpenid();
    }else if($state === "scopeUserinfo") {
        echo $webToken->getUserInfo();
    }
    ?>
    ```

3. 在前端处理用户登录的公共脚本userLogin.js，以openid唯一标识用户
   - $userLogin(function(openID) {});
   - 使用说明：在页面中引入该脚本，传入回调函数统一处理openid，并且执行顺序优先于页面所有业务逻辑，也就是说传入的回调函数是页面业务逻辑的执行入口
   - 引入cookie机制：将openid缓存到cookie进行标识用户登录状态，如果检测到cookie有openid，开始执行页面的业务逻辑，如果检测到cookie不存在openid，需要进行授权
   - 授权流程：先静默授权拿openid，根据openid进行用户查重，如果用户存在说明cookie被清，则将openid重新缓存到cookie；如果用户不存在说明该用户首次访问，进行授权登录，拿用户的个人信息保存到数据库，并且将openid重新缓存到cookie；由于授权后回调url会带上code字段，针对这种情况在openid缓存进cookie后，跳转回页面干净的url

    ```javascript
    (function(window) {
    	"use strict";
    
        var APPID = "your appid", // 公众号的应用ID，查看(公众平台-开发-基本配置-开发者ID)
            BASE_STATE = "scopeBase", // 标识静默授权状态
            USERINFO_STATE = "scopeUserinfo", // 标识授权登录状态
            openid = store.get("openid"), // 使用store.js，查看(https://github.com/amenrun/store.js)
            code = getUrlParam("code"), // 用于判断用户是否开始授权
            state = getUrlParam("state"), // 用于区分静默授权还是授权登录状态
            pageURL = getClearURL(); // 回调页(拿到页面干净的url地址，不包括code等字段)
    
        window.$userLogin = function(call) {
            if(openid){
                // alert("检测cookie有openid，开始执行业务"); // --->>>debug
                call && call(openid);
            }else{
                if(!code) {
                    // alert("检测cookie没有openid，开始授权，先进行静默授权"); // --->>>debug
                    location.href = "https://open.weixin.qq.com/connect/oauth2/authorize?" +
                        "appid=" + APPID +
                        "&redirect_uri=" + encodeURI(pageURL) +
                        "&response_type=code&scope=snsapi_base&state=" + BASE_STATE +
                        "#wechat_redirect";
                }else{
                    if(state === "scopeBase") {
                        // alert("通过静默授权，开始用户查重"); // --->>>debug
                        checkDB(function() { // 用户不存在，进行授权登录
                            location.href = "https://open.weixin.qq.com/connect/oauth2/authorize?" +
                                "appid=" + APPID +
                                "&redirect_uri=" + encodeURI(pageURL) +
                                "&response_type=code&scope=snsapi_userinfo&state=" + USERINFO_STATE +
                                "#wechat_redirect";
                        }, function(openid) { // 用户存在，缓存openid到cookie
                            store.set("openid", openid);
                            location.href = pageURL;
                        });
                    }else if(state === "scopeUserinfo") {
                        // alert("首次登陆，需要保存您的微信个人资料"); // --->>>debug
                        saveDB(function(openid) { // 个人资料保存成功，缓存openid到cookie
                            store.set("openid", openid);
                            location.href = pageURL;
                        });
                    }
                }
            }
        };
    
        // 静默授权，拿到openid，检查数据库是否存在该用户
        function checkDB(not_existCall, existCall) {
            $.get("api/wxWebToken.php", {code: code, state: state}, function(openid) { // zepto的get、ajax
        		$.post("用户查重接口", {openid: openid}, function(r) {
        			if(用户不存在) {
        				not_existCall && not_existCall();
        			}else if(用户存在) {
        				existCall && existCall(openid);
        			}
        		});
            });
        }
        // 获取用户的个人信息，保存到数据库
        function saveDB(callback) {
            $.get("api/wxWebToken.php", {code: code, state: state}, function(r) {
                var data = JSON.parse(r);
                $.post("新增用户接口", {
                    "openid": data.openid,
                    "nickname": data.nickname,
                    "sex": data.sex,
                    "city": data.city,
                    "country": data.country,
                    "province": data.province,
                    "language": data.language,
                    "headimgurl": data.headimgurl,
                    "unionid": data.unionid,
                    "privilege": data.privilege
                }, function(r) {
                	callback && callback(data.openid);
                });
            });
        }
    
    	/**
    	 * 当前url指定查询字段的值，没有则返回null
    	 * @param name 查询字段名
    	 * @returns {string}
    	 */
    	function getUrlParam(name) {
    	    var reg = new RegExp('(^|&)' + name + '=([^&]*)(&|$)'),
    	        r = location.search.substr(1).match(reg);
    	    if(r !== null) {
    	        return unescape(r[2]);
    	    }
    	    return null;
    	}
    	/**
    	 * 清除进行授权后url地址附带的code字段等，获取干净的url
    	 * @returns {string}
    	 */
    	function getClearURL() {
    	    var url = location.href, // 获取当前页面的url地址
    	        i = url.indexOf("code"); // 获取code查询字段的起始位置
    	    if(i !== -1) {
    	        return url.replace(url.substr(i-1), ""); // 从code查询字段位置开始，包括code查询字段前面的“&”字符也去掉
    	    }
    	    return url;
    	}
    })(window);
    ```