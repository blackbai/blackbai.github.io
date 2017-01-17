---
title: 使用redis实现互粉功能
date: 2017-01-17 15:11:46
tags: [redis,php,ci]
categories: "redis"
excerpt: 123123132123
---

# 使用redis实现互粉

> 最近在写api的时候要实现一个相互关注的功能，发现如果用mysql做查询不是很理想，
  所以想能不能用redis来实现这个功能，网上一搜有很多实现的方法，结合网上的博文，
  实现了自己的功能。
  
## 1.数据库实现

一下是数据库的代码，通过保存用户的id和关注对象的id以及关注状态来判断用户的关注列
表和粉丝列表，通过联查获取用户的基本信息，入头像、名称。

   ```
   DROP TABLE IF EXISTS `shc_sns`;
     CREATE TABLE `shc_sns` (
       `sns_id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'id值',
       `sns_frommid` int(11) NOT NULL COMMENT '会员id',
       `sns_tomid` int(11) NOT NULL COMMENT '朋友id',
       `sns_addtime` int(11) NOT NULL COMMENT '添加时间',
       `sns_followstate` tinyint(1) NOT NULL DEFAULT '1' COMMENT '关注状态 1为单方关注 2为双方关注',
       PRIMARY KEY (`sns_id`),
       KEY `FROMMID` (`sns_frommid`) USING BTREE,
       KEY `TOMID` (`sns_tomid`,`sns_frommid`) USING BTREE,
       KEY `FRIEND_IDS` (`sns_tomid`) USING BTREE
     ) ENGINE=InnoDB AUTO_INCREMENT=196 DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC COMMENT='好友数据表';
    ```
## 2.使用redis实现

redis提供了很多的类型，这里我们使用'zadd'添加有序集合来实现。如果不了解'zadd'命令的
可以点击查看这个[redis zdd命令](http://www.yiibai.com/redis/sorted_sets_zadd.html)

### 1.关注
 关注分为2个步骤，第一部是把对方加入到自己的关注list里面，第二步是将我写入到对方
 的粉丝中，这里score是通过时间戳生成的方便之后做分页查询，代码如下
	```
     public function addFollowsFansById($data){
             if(empty($data)) return false;
             $data['sns_addtime'] = time();
             if($data['sns_followstate'] == 1){//单方面关注
                 $result = $this->addFollowFan($data);//添加到数据库
                 if($result){//添加到有序集合
                     $this->cache->redis->zAdd('shc_follow:'.$data['sns_frommid'],time(),$data['sns_tomid']);
                     $this->cache->redis->zAdd('shc_fans:'.$data['sns_tomid'],time(),$data['sns_frommid']);
                 }
                 return true;
             }elseif($data['sns_followstate'] == 2){//相互关注
                 $this->db->trans_begin();
                 $this->addFollowFan($data);
                 $this->editFollowFans(['sns_frommid'=>$data['sns_tomid'],'sns_tomid'=>$data['sns_frommid']],['sns_followstate'=>2]);
                 if($this->db->trans_status() === false){
                     $this->db->trans_rollback();
                     return false;
                 }else{
                     $this->db->trans_commit();
                     $this->cache->redis->zAdd('shc_follow:'.$data['sns_frommid'],time(),$data['sns_tomid']);
                     $this->cache->redis->zAdd('shc_fans:'.$data['sns_tomid'],time(),$data['sns_frommid']);
                     return true;
                 }
             }else{
                 return false;
             }
         }
	```
         
### 2.取消关注
取消关注也分为2个步骤，第一步从我的关注列表中删除对方，第二步从对方的粉丝列表中
删除我，这里用redis的'zrem'来实现从有序集合中删除记录，[redis zrem命令](https://redis.io/commands/zrem)
```
    public function deleteFollowFansById($user_id,$follow_id,$state){
            if($state == 1){
                $result = $this->deleteFollowFans(['sns_frommid'=>$user_id,'sns_tomid'=>$follow_id]);
                if($result){//缓存删除关注
                    $this->cache->redis->zRem('shc_follow:'.$user_id,$follow_id);
                    $this->cache->redis->zRem('shc_fans:'.$follow_id,$user_id);
                }
                return true;
            }elseif($state == 2){
                //开启事务
                $this->db->trans_begin();
                $this->deleteFollowFans(['sns_frommid'=>$user_id,'sns_tomid'=>$follow_id]);
                $this->editFollowFans(['sns_frommid'=>$follow_id,'sns_tomid'=>$user_id],['sns_followstate'=>1]);
                if($this->db->trans_status() === false){
                    $this->db->trans_rollback();
                    return false;
                }else{
                    $this->cache->redis->zRem('shc_follow:'.$user_id,$follow_id);
                    $this->cache->redis->zRem('shc_fans:'.$follow_id,$user_id);
                    $this->db->trans_commit();
                    return true;
                }
            }else{
                return false;
            }
        }
```
### 3.查看关注列表、粉丝列表
查看关注、粉丝列表原理一样，用redis的'zrange'，'zrange'的形式如下

    ZRANGE KEY START STOP

所以我们传入要获取的开始条数'curreng_page'，以及要获取的条数 = 'current_page+page_size-1'，
这里因为redis开始是1所以要-1，而数据库是从0开始查询，代码如下：
[redis zrange命令](http://www.yiibai.com/redis/sorted_sets_zrange.html)
      ```
	  public function getFollowsListById($user_id,$limit=array()){
            $follow_list = $this->cache->redis->zRevRange('shc_follow:'.$user_id,$limit['current_page'],$limit['current_page']+$limit['page_size']-1);
            if($follow_list){
                $member_list = $this->Member_Model->getMemberListByIdS($follow_list,'user_id,user_nickname,user_portrait');
                if($member_list) return $member_list;
            }else{
                return $this->getFollowsListByDb(['a.sns_frommid'=>$user_id],'b.user_id,b.user_nickname,b.user_portrait',$limit);
            }
        }
		```

### 4.关注、粉丝数量
通过redis的'zcard'命令来返回有序集合的数量，也就是粉丝、关注的数量

### 5.关注状态
关注状态分为3中类型，我当方面关注、你单方面关注、你我相互关注，所以很好理解，只要判断有序集合中是否存在记录就可以了，
这种可以用到redis的'zcore'命令，
	```
    //我关注你，你没关注我
    ZSCORE 1:follow 2 #ture
    ZSCORE 1:fans 2 #false
    
    //你关注我，我没关注你
    ZSCORE 1:follow 2 #false
    ZSCORE 1:fans 2 #true
    
    //相互关注
    ZSCORE 1:follow 2 #true
    ZSCORE 1:fans 2 #true
	```

## 总结
通过redis我们实现了互粉的功能，类似的也可以实现评论的功能，这样我们就能够缓减mysql的压力。



