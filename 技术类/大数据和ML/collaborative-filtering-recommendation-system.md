个人博客原文：[协同过滤Item-based算法实现电影推荐系统](http://blog.taohuawu.club/article/item-based-movie-recommendation)

>摘要: 采用离线式计算推荐给每位用户的电影，采用Item-based算法并做了适当修改，
>主要分两部分： 
>1. 计算电影的相似度：利用调整的余弦相似度计算方法； 
>2. 相似度加权求和：使用用户已打分的电影的分数进行加权求和，权值为用户未打分的各电影与打分的各电影的相似度，然后对所有相似度的和求平均。

# 系统详细设计
## 离线计算推荐电影模块
### 系统所用算法
本系统采用协同过滤（Collaborative Filtering）推荐算法。协同过滤推荐算法分为预测过程和推荐过程，其包括Item-based算法和User-based算法，但经查阅相关资料发现User-based算法存在两个问题：
1. 数据的稀疏性：一个大型的电影推荐系统会有大量的电影信息，用户已打分的电影可能只占总量的很少一部分，不同用户之间电影打分的重叠性较低，导致算法无法找到一个兴趣用户；
2. 算法的扩展性：最近邻算法的计算量会随着用户和电影信息数量的增加而增加，不适合信息量大的情况。所以本系统采用了Item-based协同过滤算法，并对其做了适当修改。

### 计算过程
计算过程分两步： 
1. 计算电影相似度：利用调整的余弦相似度计算方法，公式如下图： 
  ![余弦相似度算法](http://118.126.91.74/upload/2018/04/5pls8qktt4g5spt65pb9c5c2g0.jpg)
2. 电影相似度加权求和：使用用户已打分电影的分数进行加权求和，权值为用户未打分的各电影与打分的各电影的相似度，然后求平均，公式如下图： 
  ![加权求和](http://118.126.91.74/upload/2018/04/t4hfn48t9miq9q9m5i9uchjhe4.jpg)
  以上部分均采用了多线程技术计算，值得注意的是相似度计算存在很大的冗余，必须去除冗余计算，不然计算量很大，这需要在数据库中建立一个sim表来记录已经计算的的sim(i,j),当下次需要计算sim(i,j)或sim(j,i)可直接从数据库读取而无需重复计算。并且我认为如果R（u，N）值如果比较低，说明用户不喜欢此电影，这条相似度信息可以忽略，这需要设置一个打分阀值。

### 计算模块流程图： 
由于计算量较大，计算模块采用了多线程技术，来提升系统运行效率，如下图为计算模块主线程和子线程的流程图，如图ａ，图ｂ： 
![图a](http://118.126.91.74/upload/2018/04/51gop0qovehi4o5shu7hutol76.jpg)


![图b](http://118.126.91.74/upload/2018/04/ujl3g7654oh09r93k5edf9oth0.jpg)


### 关键代码分析
计算模块采用多线程技术，主线程通过创建子线程来实现大量数据的计算，其中线程状态的监测很有必要，如下面为主线程实现代码：
```java
sql="SELECT uid FROM user group by uid";
		uidItems=uiddb.executeQuery(sql);	
		for(int i=0;i<threadNum;i++){
			thCom[i]=new Thread(new thComp(uidItems,i));	
			thCom[i].start();
		}//for i
		boolean thFinished=false;
		while(!thFinished){
			try{
				Thread.sleep(sleepTime);
			}catch(Exception e){//try
				e.printStackTrace();
			}//catch();
			thFinished=true;
			for(int i=0;i<threadNum;i++){
				if(thCom[i].isAlive()){
					//System.out.println(thCom[i].getName()+":"+thCom[i].getState());
					thFinished=false;
					break;
				}//if
			}//for i
		}//while(!thFinished)
```

由于子线程是从主线程创建的ResultSet中读取数据，这就需要子线程互斥的访问临界资源ResultSet，如下面代码所示：
```java
public synchronized int getUserID(ResultSet uidItems){
		int uid=0;//0 shows that no usrid needs to process
		try{
			if(uidItems.next()){
				uid=uidItems.getInt("uid");
			}//if
		}catch(Exception e){
			e.printStackTrace();
		}//catch
		return uid;
	}//getUserID()
```
在为用户A计算要推荐的电影，计算电影i与电影j的相似度sim(i,j)时，可能在为用户B计算要推荐的电影已经计算过，如果再计算一次，会造成计算的冗余，降低了计算效率，使系统性能下降，这需要将将计算的sim(i,j)保存下来，然后需要计算sim(i,j)先判断是否已经计算过sim(i,j)或sim(j,i)，如果计算过直接读取，否则计算，本系统中是将sim(i,j)的计算值保存到数据库的sim表中，主要实现代码如下：
```java
String sql="SELECT sim FROM sim WHERE midi='"+midi+"' and midj='"+midj+"' or midi='"+midj+"' and midj='"+midi+"'";	
		uidItems=uiddb.executeQuery(sql);
		//select and fllaowing insert may have problems
		try{
			if(uidItems.next()){
				return uidItems.getDouble("sim");
			}//if(uidItems.next())
		}catch(Exception e){
			e.printStackTrace();
		}//catch()
	public synchronized void insertSim(String sql){
		//here may have some problems
		recSimdb.executeUpdate(sql);
	}//insertSim
```
## 浏览推荐电影模块
### 浏览用户推荐电影流程图
![浏览电影流程图](http://118.126.91.74/upload/2018/04/pdv2aapkg2i27o713mu8cdqvbm.jpg)

### 关键代码分析 　　
该模块主要是按推荐指数降序排序显示推荐给每个用户的电影，来满足用户的娱乐需求，提高用户的满意率，此功能主要代码如下所示：
```java
public synchronized int getUserID(ResultSet uidItems){
		int uid=0;//0 shows that no usrid needs to process
		try{
			if(uidItems.next()){
				uid=uidItems.getInt("uid");
			}//if
		}catch(Exception e){
			e.printStackTrace();
		}//catch
		return uid;
	}//getUserID()
```
## 数据库设计模块
由于电影和用户信息数据量比较大，本系统将数据文件中的数据导入到mysql数据库中，同时增加了一些额外数据表来满足系统的要求，比如保存用户平均打分的avgrating表和降低计算电影相似性冗余的sim表。在经常查询的字段添加了索引，以提高查询速度，在group by字段也添加了索引。下面简单介绍下数据库中各个表结构如下几张图所示:

```sql
CREATE TABLE IF NOT EXISTS `item` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `mid` int(11) NOT NULL,
  `title` varchar(255) DEFAULT NULL,
  `releasedate` varchar(50) DEFAULT NULL,
  `videoreleasedate` varchar(50) DEFAULT NULL,
  `imdburl` varchar(255) DEFAULT NULL,
  `unknown` int(11) DEFAULT '0',
  `action` int(11) DEFAULT '0',
  `adventure` int(11) DEFAULT '0',
  `animation` int(11) DEFAULT '0',
  `children` int(11) DEFAULT '0',
  `comedy` int(11) DEFAULT '0',
  `crime` int(11) DEFAULT '0',
  `documentary` int(11) DEFAULT '0',
  `drama` int(11) DEFAULT '0',
  `fantasy` int(11) DEFAULT '0',
  `filmnoir` int(11) DEFAULT '0',
  `horror` int(11) DEFAULT '0',
  `musical` int(11) DEFAULT '0',
  `mystery` int(11) DEFAULT '0',
  `romance` int(11) DEFAULT '0',
  `scifi` int(11) DEFAULT '0',
  `thriller` int(11) DEFAULT '0',
  `war` int(11) DEFAULT '0',
  `western` int(11) DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `item_mid` (`mid`)
) ENGINE=InnoDB  DEFAULT CHARSET=gbk AUTO_INCREMENT=11775 ;
```

```sql
CREATE TABLE IF NOT EXISTS `avgrating` (
  `usrid` int(11) NOT NULL,
  `avgRating` double DEFAULT NULL,
  UNIQUE KEY `avgrating_usrid` (`usrid`),
  UNIQUE KEY `avgRating_usridAvgRating` (`usrid`,`avgRating`)
) ENGINE=InnoDB DEFAULT CHARSET=gbk;
```

```sql
CREATE TABLE IF NOT EXISTS `genre` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `genre` varchar(50) NOT NULL,
  `mid` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=gbk AUTO_INCREMENT=151 ;
```

```sql
CREATE TABLE IF NOT EXISTS `info` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `users_num` int(11) DEFAULT NULL,
  `items_num` int(11) DEFAULT NULL,
  `ratings_num` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=gbk AUTO_INCREMENT=1 ;
```

```sql
CREATE TABLE IF NOT EXISTS `rating` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `usrid` int(11) DEFAULT NULL,
  `itemid` int(11) DEFAULT NULL,
  `rating` int(11) DEFAULT NULL,
  `timestamp` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `rating_uidiid` (`usrid`,`itemid`)
) ENGINE=InnoDB  DEFAULT CHARSET=gbk AUTO_INCREMENT=500001 ;
```




## 协同过滤Item-based算法
（1）相似度计算 
Item-based算法首选计算物品之间的相似度，计算相似度的方法有以下几种： 
1. 基于余弦（Cosine-based）的相似度计算，通过计算两个向量之间的夹角余弦值来计算物品之间的相似性，公式如下：
  ![Cosine-based](http://blog.taohuawu.club/upload/2018/05/odp6gi8jjqhfmp5ood28rgsoqd.png)
  其中分子为两个向量的内积，即两个向量相同位置的数字相乘。 
2. 基于关联（Correlation-based）的相似度计算，计算两个向量之间的Pearson-r关联度，公式如下：
  ![Correlation-based](http://blog.taohuawu.club/upload/2018/05/7b0gr2fm3ojqbqj994t5gpmcod.png)
  其中表示用户u对物品i的打分，表示第i个物品打分的平均值。 
3. 调整的余弦（Adjusted Cosine）相似度计算，由于基于余弦的相似度计算没有考虑不同用户的打分情况，可能有的用户偏向于给高分，而有的用户偏向于给低分，该方法通过减去用户打分的平均值消除不同用户打分习惯的影响，公式如下：
  ![Adjusted Cosine](http://blog.taohuawu.club/upload/2018/05/r409dr2n5ggfbr09jr33bb5km3.png)
  其中表示用户u打分的平均值。

（2）预测值计算 
根据之前算好的物品之间的相似度，接下来对用户未打分的物品进行预测，有两种预测方法：

1. 加权求和
  用过对用户u已打分的物品的分数进行加权求和，权值为各个物品与物品i的相似度，然后对所有物品相似度的和求平均，计算得到用户u对物品i打分，公式如下：
  ![加权求和](http://blog.taohuawu.club/upload/2018/05/o1o1viaerijhfqnadaria9mf4f.png)

其中为物品i与物品N的相似度，为用户u对物品N的打分。
2. 回归 
  和上面加权求和的方法类似，但回归的方法不直接使用相似物品N的打分值，因为用余弦法或Pearson关联法计算相似度时存在一个误区，即两个打分向量可能相距比较远（欧氏距离），但有可能有很高的相似度。因为不同用户的打分习惯不同，有的偏向打高分，有的偏向打低分。如果两个用户都喜欢一样的物品，因为打分习惯不同，他们的欧式距离可能比较远，但他们应该有较高的相似度。在这种情况下用户原始的相似物品的打分值进行计算会造成糟糕的预测结果。通过用线性回归的方式重新估算一个新的值，运用上面同样的方法进行预测。重新计算的方法如下：
  ![回归](http://blog.taohuawu.club/upload/2018/05/29o1f1et4oiogrf6q540c3rmeo.png)

### 算法实现（java）：
```java
import java.sql.ResultSet;
import movierec.DB;
import movierec.FUN;
import movierec.SYS;
public class ItemBasedSim {
	/**
	 * @param args
	 */
	public int threadNum=5;
	public int sleepTime=3000;//ms
	public double thresholdRating=2.5;
	public int recNum=50;//the num of movies to recommend,which need to be computed
	public int RuNNum=20;//
	public Thread thCom[];
	DB recSimdb=null;
	//public static void main(String[] args) {
		// TODO Auto-generated method stub
		//System.out.println("test");
		/*
		long startTime = System.currentTimeMillis();
		System.out.println(fun.getDateTime()+" starts...");
		itemBasedSim ibs=new itemBasedSim (5);
		//ibs.sim(3,6);
		//System.out.println("sim:"+ibs.sim(5,9));
		ibs.compRecDeg();
		//ibs.compRecDegUI(1,1);
		long finishTime = System.currentTimeMillis();
		System.out.println(fun.getDateTime()+" finishs,spent:"+(finishTime-startTime)+"ms");
		*/
		//System.out.println("spent:"+(finishTime-startTime)+"ms");
		/*startTime = System.currentTimeMillis();
		ibs.compRecDegUI(2,1);
		finishTime = System.currentTimeMillis();
		System.out.println("spent:"+(finishTime-startTime)+"ms");*/
	//}//main()
	public ItemBasedSim(int thNum){
		threadNum=thNum;
		thCom=new Thread[threadNum];
	}//itemBasedSim()
	public void compRecDeg(){
		//SELECT avg(rating) FROM rating,item WHERE mid=itemid and war=1 group by usrid
		DB uiddb=new DB();	
		recSimdb=new DB();
		ResultSet uidItems=null;	
		String sql="delete FROM avgrating";
		uiddb.executeUpdate(sql);
		sql="insert into avgrating SELECT usrid,avg(rating) avgRating FROM rating group by usrid";
		uiddb.executeUpdate(sql);
		//sql="delete FROM sim";
		//uiddb.executeUpdate(sql);
		sql="delete FROM recom";
		uiddb.executeUpdate(sql);
		sql="SELECT uid FROM user group by uid";
		uidItems=uiddb.executeQuery(sql);	
		for(int i=0;i<threadNum;i++){
			thCom[i]=new Thread(new thComp(uidItems,i));	
			thCom[i].start();
		}//for i
		boolean thFinished=false;
		while(!thFinished){
			try{
				Thread.sleep(sleepTime);
			}catch(Exception e){//try
				e.printStackTrace();
			}//catch();
			thFinished=true;
			for(int i=0;i<threadNum;i++){
				if(thCom[i].isAlive()){
					//System.out.println(thCom[i].getName()+":"+thCom[i].getState());
					thFinished=false;
					break;
				}//if
			}//for i
		}//while(!thFinished)
		recSimdb.close_state();
		recSimdb.close_connect();
		uiddb.close_result();
		uiddb.close_state();
		uiddb.close_connect();
	}//compRecDeg()
	public double compRecDegUI(int uid,int iid,int thID,DB mrdb,DB mydb,DB uiddb){
		//db mrdb=new db();
		String mrsql="SELECT itemid,rating FROM rating WHERE usrid='"+uid+"'";
		mrsql+=" and rating>='"+this.thresholdRating+"' limit 0,"+this.RuNNum;
		//need to 
		ResultSet mrItems=null;
		double simUI=0;
		double fz=0;
		double fm=0;
		double pui=0;
		//System.out.println("uid="+uid+"  , itemid= "+iid+" , threadID="+thID);
		mrItems=mrdb.executeQuery(mrsql);
		try{
			while(mrItems.next()){				
				simUI=sim(iid,mrItems.getInt("itemid"),thID,mydb,uiddb);
				fz+=simUI*mrItems.getInt("rating");
				fm+=Math.abs(simUI);
				//System.out.println(uid+","+iid+"----------2");
			}//while(uidItems.next())
			if(fm!=0)
				pui=fz/fm;
			else
				pui=0;
			//System.out.println("----------3");
		}catch(Exception e){
			e.printStackTrace();
		}//catch()
		return pui;
	}//comRecDegUI()
	public double sim(int midi,int midj,int thID,DB mydb,DB uiddb){
		//db mydb=new db();
		//db uiddb=new db();
		ResultSet mItems=null;
		ResultSet uidItems=null;
		String sql="SELECT sim FROM sim WHERE midi='"+midi+"' and midj='"+midj+"' or midi='"+midj+"' and midj='"+midi+"'";	
		uidItems=uiddb.executeQuery(sql);
		//select and fllaowing insert may have problems
		try{
			if(uidItems.next()){
				return uidItems.getDouble("sim");
			}//if(uidItems.next())
		}catch(Exception e){
			e.printStackTrace();
		}//catch()
		///String sqlUid="SELECT usrid FROM rating GROUP BY usrid ";
		String sqlUid="SELECT usrid,avgRating FROM avgrating ";
		//maybe there are users who has not commented any movie
		uidItems=uiddb.executeQuery(sqlUid);	
		int uid=0;
		double ru=0;
		double rui=0;
		double ruj=0;
		double mid1=0;//(rui-ru)(ruj-ru)
		double mid2=0;//pow((rui-ru),2);
		double mid3=0;//pow((ruj-ru),2);
		try{	
			while(uidItems.next()){	
				ru=0;
				rui=0;
				ruj=0;
				uid=uidItems.getInt("usrid");
				ru=uidItems.getDouble("avgRating");		
				sql="SELECT rating FROM rating WHERE usrid='"+uid+"' AND itemid ='"+midi+"'";
				mItems= mydb.executeQuery(sql);
				if(mItems.next()){
					rui=mItems.getDouble("rating");
				}//if(mitems.next())			
				sql="SELECT rating FROM rating WHERE usrid='"+uid+"' AND itemid ='"+midj+"'";
				mItems= mydb.executeQuery(sql);	
				if(mItems.next()){
					ruj=mItems.getDouble("rating");
				}//if(mitems.next())
				//System.out.println("RU="+ru);
				//System.out.println("RUI="+rui);
				//System.out.println("RUJ="+ruj);
				mid1+=(rui-ru)*(ruj-ru);
				mid2+=Math.pow(rui-ru, 2);
				mid3+=Math.pow(ruj-ru,2);
			}//while(uidItems.next())
			mid2=Math.sqrt(mid2)+Math.sqrt(mid3);
			if(mid2!=0){
				mid3=mid1/mid2;
			}else
				mid3=0;
		}catch(Exception e){
			e.printStackTrace();
		}//catch
		sql="INSERT INTO sim (midi, midj, sim,timestamp) VALUES ('"+midi+"','"+midj+"','"+mid3+"','";
		sql+=System.currentTimeMillis()+thID+Math.random()+"')";
		//mydb.executeUpdate(sql);
		//insert may have problems ,because some select no sim at the same time
		//but have unique index
		try{
			insertSim(sql);
		}catch(Exception e){
			//
		}//catch
		return mid3;		
	}//sim()
	public synchronized int getUserID(ResultSet uidItems){
		int uid=0;//0 shows that no usrid needs to process
		try{
			if(uidItems.next()){
				uid=uidItems.getInt("uid");
			}//if
		}catch(Exception e){
			e.printStackTrace();
		}//catch
		return uid;
	}//getUserID()
	public synchronized void insertRecom(String sql){
		recSimdb.executeUpdate(sql);
	}//insertRecom
	public synchronized void insertSim(String sql){
		//here may have some problems
		recSimdb.executeUpdate(sql);
	}//insertSim
	
	class thComp implements Runnable{	
		ResultSet uidItems=null;	
		int thID=0;	
		int gNum[]=new int[SYS.genre.length];
		int gNumS=0;
		public void run(){
			int uid=0;
			int mid=0;
			String isql="";
			String rsql="";
			//String orderCol[]={"id","usrid","itemid","rating","timestamp"};
			double recDeg=0;
			DB middb=new DB();
			DB mrdb=new DB();
			DB mydb=new DB();
			DB uiddb=new DB();
			ResultSet midItems=null;
			int midNum=0;
			int i=0;
			int hasRecIid[] = new int[SYS.recNum*2+5];
			int hRNum=0;
			while(0!=(uid=getUserID(uidItems))){
				//here should compute the user's intersting to decide the num of each 
				//kind to recommend
			    hRNum=0;
				gNumS=0;
				for(i=0;i<SYS.genre.length;i++){		
					gNum[i]=0;
					isql="SELECT avg(rating) avgR FROM rating,item WHERE mid=itemid and "+SYS.genre[i]+"=1 and usrid='"+uid+"'";
					midItems=middb.executeQuery(isql);
					try{
						if(midItems.next())
						gNum[i]=(int) midItems.getDouble("avgR");
					}catch(Exception e){
						e.printStackTrace();
					}//catch
					gNumS+=gNum[i];
				}//for i
				if(gNumS>0){				
					for(i=0;i<SYS.genre.length;i++){
						midNum=(int) (gNum[i]/(gNumS*1.0)*SYS.recNum*2);
						if(midNum>=1){
							isql="SELECT itemid FROM rating,item WHERE mid=itemid and "+SYS.genre[i]+"=1 and usrid='"+uid+"' and rating>='"+thresholdRating;
							isql+="' group by itemid order by rating desc limit 0,"+midNum;
							//System.out.println(isql);
							//各类间可能会有重复的item							
							midItems=middb.executeQuery(isql);
							try{
								while(midItems.next()){
									//System.out.println("----");
									recDeg=0;
									mid=midItems.getInt("itemid");
									if(hasRec(hasRecIid,hRNum,mid))
										continue;
									hasRecIid[hRNum++]=mid;
									//System.out.println(thID+":("+hRNum+"):"+uid+","+mid);
									recDeg=compRecDegUI(uid,mid,thID,mrdb,mydb,uiddb);//very cost time
									rsql="INSERT INTO recom (id, uid, mid, recdeg) VALUES (NULL,'"+uid+"','"+mid+"','"+recDeg+"')";
									//System.out.println(rsql);
									insertRecom(rsql);
								}//while(iidItems.next())
							}catch(Exception e){
								e.printStackTrace();
							}//catch()
						}//if(midNum>=1)
					}//for
				}else{
					isql="SELECT itemid FROM rating  where usrid!='"+uid+"' and rating>="+thresholdRating+" group by itemid order by rating desc";
					isql+=" limit 0,"+SYS.recNum*2;
					midItems=middb.executeQuery(isql);
					try{
						while(midItems.next()){
							recDeg=0;
							mid=midItems.getInt("itemid");
							recDeg=compRecDegUI(uid,mid,thID,mrdb,mydb,uiddb);
							rsql="INSERT INTO recom (id, uid, mid, recdeg) VALUES (NULL,'"+uid+"','"+mid+"','"+recDeg+"')";
							//System.out.println(rsql);
							insertRecom(rsql);
						}//while(iidItems.next())
					}catch(Exception e){
						e.printStackTrace();
					}//catch()
				}//else
			}//while(0!=(uid=getUserID(uidItems)))
			middb.close_result();
			middb.close_state();
			middb.close_connect();
			uiddb.close_result();
			uiddb.close_state();
			uiddb.close_connect();
			mrdb.close_result();
			mrdb.close_state();
			mrdb.close_connect();
		}//run()
		public thComp(ResultSet uidItems,int thID){
			this.uidItems=uidItems;
			this.thID=thID;
		}//thComp
		public boolean hasRec(int hasRecIid[],int hRNum,int iid){
			boolean flag=false;
			for(int i=0;i<hRNum;i++)
				if(hasRecIid[i]==iid){
					flag=true;
					break;
				}//if(hasRecIid[i]==iid)
			return flag;
		}
	}//class cdatatodb
}
```
**全部代码可在博主的github上查看：[https://github.com/panjf2000/mvrecmd](https://github.com/panjf2000/mvrecmd)**