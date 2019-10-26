# 3D Game Programming & Design：物理系统与碰撞
## 作业要求
- 提交要求：
- 游戏设计要求：
创建一个地图和若干巡逻兵(使用动画)；
每个巡逻兵走一个3~5个边的凸多边型，位置数据是相对地址。即每次确定下一个目标位置，用自己当前位置为原点计算；
巡逻兵碰撞到障碍物，则会自动选下一个点为目标；
巡逻兵在设定范围内感知到玩家，会自动追击玩家；
失去玩家目标后，继续巡逻；
计分：玩家每次甩掉一个巡逻兵计一分，与巡逻兵碰撞游戏结束；
- 程序设计要求：
	- 必须使用订阅与发布模式传消息
subject：OnLostGoal
Publisher: ?
Subscriber: ?
	- 工厂模式生产巡逻兵
- 友善提示1：生成 3~5个边的凸多边型
随机生成矩形
在矩形每个边上随机找点，可得到 3 - 4 的凸多边型
5 ?
- 友善提示2：参考以前博客，给出自己新玩法
##  模式分析：订阅与发布模式
在“发布者-订阅者”模式中：
>称为发布者的消息发送者不会将消息编程为直接发送给称为订阅者的特定接收者。这意味着发布者和订阅者不知道彼此的存在。存在第三个组件，称为代理或消息代理或事件总线，它由发布者和订阅者都知道，它过滤所有传入的消息并相应地分发它们。

换句话说，pub-sub是用于在不同系统组件之间传递消息的模式，而这些组件不知道关于彼此身份的任何信息。经纪人如何过滤所有消息？实际上，有几个消息过滤过程。最常用的方法有：基于主题和基于内容的。

**发布/订阅模式的有点非常明显：**

- 时间上的解耦：发布者/订阅者在大多情况下是异步方式（使用消息队列）。
- 对象之间的解耦：在Publisher / Subscriber模式中，组件是松散耦合的，而不是Observer模式。


## 代码分析

本次作业[参考](https://blog.csdn.net/qq_33000225/article/details/70045292)了前辈的博客，设计的UML图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026211951346.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDM3NzY5MQ==,size_16,color_FFFFFF,t_70)
- 订阅与发布模式的利用
利用订阅和发布模式传信息，实现不同预制之间信息的传递，也即实现游戏中GameStatus（游戏状态）和CanvasStatus（分数和游戏界面显示） 的交流，信息包括玩家得分和游戏结束信息。
	```javascript
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	using Com.Patrols;
	
	public class GameStatus: MonoBehaviour {
	    public delegate void GameScoreAction();
	    public static event GameScoreAction myGameScoreAction;
	
	    public delegate void GameOverAction();
	    public static event GameOverAction myGameOverAction;
	
	    private SceneController scene;
		private int canMove = 1;
	
	    void Start () {
	        scene = SceneController.getInstance();
	        scene.setGameStatus(this);
	    }
		
		void Update () {
			
		}
	
	    //hero逃离巡逻兵，得分
	    public void heroEscapeAndScore() {
	        myGameScoreAction();
			canMove = 1;
	    }
	
	    //巡逻兵捕获hero，游戏结束
	    public void patrolHitHeroAndGameover() {
	        myGameOverAction();
			canMove = 0;
	    }
	}
	```
	由于此次作业的得分/游戏结束均只需改变得分数or显示"Game Over!"，所以我在挂载在UI.text上的脚本CanvasStatus订阅两种信息gameOver和gameScore即可，这样就可以在触发玩家得分、游戏结束时，调用GetComponent< Text >自己改变文字内容。
	```javascript
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	using UnityEngine.UI;
	
	//----------------------------------
	// 此脚本加在text上
	//----------------------------------
	
	public class CanvasStatus : MonoBehaviour {
	    private int score = 0;
		private int textType;  
	
		void Start () {
			distinguishText();
		}
		
		void Update () {
		}
	    void distinguishText() {
			if (gameObject.name.Contains ("Score"))
				textType = 0;
			else {
				Debug.Log (gameObject.name);
				textType = 1;
			}
	    }
	    void OnEnable() {
	       	GameStatus.myGameScoreAction += gameScore;
			GameStatus.myGameOverAction += gameOver;
	    }
	
	    void OnDisable() {
	        GameStatus.myGameScoreAction -= gameScore;
	        GameStatus.myGameOverAction -= gameOver;
	    }
	
	    void gameScore() {
			if (textType == 0 && this.gameObject.name.Contains("Score")) {
	            score++;
	            this.gameObject.GetComponent<Text>().text = "Score: " + score;
	        }
	    } 
	
	    void gameOver() {
			if (textType == 1) {
				this.gameObject.GetComponent<Text> ().text = "Game Over!";
	
			}
		}
	}
	```
	这种模式可以有效的实现功能的分类，降低代码耦合。
- PatrolFactory主要用来实现巡逻兵和玩家的生产
	```javascript
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	using Com.Patrols;
	
	namespace Com.Patrols {
	    public class PatrolFactory : System.Object {
	        private static PatrolFactory instance;
	        private GameObject PatrolItem;
			private GameObject HeroItem;
	
	        private Vector3[] PatrolPosSet = new Vector3[] { new Vector3(-6, 0, 16), new Vector3(-1, 0, 19),
	            new Vector3(6, 0, 16), new Vector3(-5, 0, 7), new Vector3(0, 0, 7), new Vector3(6, 0, 7)};
	
	        public static PatrolFactory getInstance() {
	            if (instance == null)
	                instance = new PatrolFactory();
	            return instance;
	        }
	
	        public void initPatrol(GameObject _PatrolItem) {
	            PatrolItem = _PatrolItem;
	        }
			public void initHero(GameObject _Hero){
				HeroItem = _Hero;
			}
	        public GameObject getPatrol() {
	            GameObject newPatrol = Camera.Instantiate(PatrolItem);
	            return newPatrol;
	        }
			public GameObject getHero(){
				GameObject newHero = Camera.Instantiate (HeroItem);
				return newHero;
			}
	        public Vector3[] getPosSet() {
	            return PatrolPosSet;
	        }
	    }
	}
	```
- 将上述脚本中的初始化载入GameModel中，在这个脚本中具体实现巡逻兵和玩家直接动作的交互。

	这里动作实现的难点在于如何实现巡逻兵的循环走动，这里主要涉及两个函数：
	addRandomMovement：当巡逻兵为非追捕状态时，添加随机方向动作
	```javascript
	 public void addRandomMovement(GameObject sourceObj, bool isActive) {
			if (ifStop == 1)
				return;
	        int index = getIndexOfObj(sourceObj);
	        int randomDir = getRandomDirection(index, isActive);
	        PatrolLastDir[index] = randomDir;
	
	        sourceObj.transform.rotation = Quaternion.Euler(new Vector3(0, randomDir * 90, 0));
	        Vector3 target = sourceObj.transform.position;
	        switch (randomDir) {
	            case Diretion.UP:
	                target += new Vector3(0, 0, 1);
	                break;
	            case Diretion.DOWN:
	                target += new Vector3(0, 0, -1);
	                break;
	            case Diretion.LEFT:
	                target += new Vector3(-1, 0, 0);
	                break;
	            case Diretion.RIGHT:
	                target += new Vector3(1, 0, 0);
	                break;
	        }
	        addSingleMoving(sourceObj, target, PERSON_SPEED_NORMAL, false);
	    }
	```
	addDirectMovement：当巡逻兵为追捕状态时，添加指向玩家的移动动作，它的实现就是用巡逻兵和玩家的位置相减得到移动方向。
		
	```javascript
	 public void addDirectMovement(GameObject sourceObj) {
			if (ifStop == 1)
				return;
	        int index = getIndexOfObj(sourceObj);
	        PatrolLastDir[index] = -2;
	
	        sourceObj.transform.LookAt(sourceObj.transform);
	        Vector3 oriTarget = myHero.transform.position - sourceObj.transform.position;
	        Vector3 target = new Vector3(oriTarget.x / 4.0f, 0, oriTarget.z / 4.0f);
	        target += sourceObj.transform.position;
	        //Debug.Log("addDirectMovement: " + target);
			addSingleMoving(sourceObj, target, PERSON_SPEED_CATCHING, true);
	    }
	
	    void addSingleMoving(GameObject sourceObj, Vector3 target, float speed, bool isCatching) {
	        this.runAction(sourceObj, CCMoveToAction.CreateSSAction(target, speed, isCatching), this);
	    }
	```
	这里要注意的是巡逻兵不能走出自己的区域，我们通过一个bool函数来做判断。
	```javascript
	 //判定巡逻兵走出了自己的区域
	    bool PatrolOutOfArea(int index, int randomDir) {
	        Vector3 patrolPos = PatrolSet[index].transform.position;
	        float posX = patrolPos.x;
	        float posZ = patrolPos.z;
	        switch (index) {
	            case 0:
	                if (randomDir == 1 && posX + 1 > FenchLocation.FenchVertLeft
	                    || randomDir == 2 && posZ - 1 < FenchLocation.FenchHori)
	                    return true;
	                break;
	            case 1:
	                if (randomDir == 1 && posX + 1 > FenchLocation.FenchVertRight
	                    || randomDir == -1 && posX - 1 < FenchLocation.FenchVertLeft
	                    || randomDir == 2 && posZ - 1 < FenchLocation.FenchHori)
	                    return true;
	                break;
	            case 2:
	                if (randomDir == -1 && posX - 1 < FenchLocation.FenchVertRight
	                    || randomDir == 2 && posZ - 1 < FenchLocation.FenchHori)
	                    return true;
	                break;
	            case 3:
	                if (randomDir == 1 && posX + 1 > FenchLocation.FenchVertLeft
	                    || randomDir == 0 && posZ + 1 > FenchLocation.FenchHori)
	                    return true;
	                break;
	            case 4:
	                if (randomDir == 1 && posX + 1 > FenchLocation.FenchVertRight
	                    || randomDir == -1 && posX - 1 < FenchLocation.FenchVertLeft
	                    || randomDir == 0 && posZ + 1 > FenchLocation.FenchHori)
	                    return true;
	                break;
	            case 5:
	                if (randomDir == -1 && posX - 1 < FenchLocation.FenchVertRight
	                    || randomDir == 0 && posZ + 1 > FenchLocation.FenchHori)
	                    return true;
	                break;
	        }
	        return false;
	    }
	```
- 玩家挂载脚本HeroStatus
	该脚本上有个standOnArea的整形变量，时刻根据玩家位置改变该变量的值。
	```javascript
	    //检测所在区域
	    void modifyStandOnArea() {
	        float posX = this.gameObject.transform.position.x;
	        float posZ = this.gameObject.transform.position.z;
	        if (posZ >= FenchLocation.FenchHori) {
	            if (posX < FenchLocation.FenchVertLeft)
	                standOnArea = 0;
	            else if (posX > FenchLocation.FenchVertRight)
	                standOnArea = 2;
	            else
	                standOnArea = 1;
	        }
	        else {
	            if (posX < FenchLocation.FenchVertLeft)
	                standOnArea = 3;
	            else if (posX > FenchLocation.FenchVertRight)
	                standOnArea = 5;
	            else
	                standOnArea = 4;
	        }
	    }
	```
	巡逻兵在Update()方法里时刻检测该值来判断玩家是否进入自己区域。若是，添加跟踪动作addDirectMovement（在上面GameModel.cs 里实现），而原动作也自动销毁。
		```javascript
		void Update () {
		        modifyStandOnArea();
			}
		```
	
  - 巡逻兵挂载脚本PatrolBehaviour
	这里有个isCatching的变量，代表巡逻兵是否处于追捕状态。若处于追捕状态，而发现玩家已不在自己的区域，说明在刚一瞬间，玩家逃离了自己区域，此时会添加随机动作，即继续巡逻。
	```javascript
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	using Com.Patrols;
	
	//----------------------------------
	// 此脚本加在巡逻兵上
	//----------------------------------
	
	public class PatrolStatus : MonoBehaviour {
	    private IAddAction addAction;
	    private IGameStatusOp gameStatusOp;
	
	    public int ownIndex;
	    public bool isCatching;    //是否感知到hero
	
	    private float CATCH_RADIUS = 3.0f;
	
	    void Start () {
	        addAction = SceneController.getInstance() as IAddAction;
	        gameStatusOp = SceneController.getInstance() as IGameStatusOp;
	
	        ownIndex = getOwnIndex();
	        isCatching = false;
	    }
		
		void Update () {
	        checkNearByHero();
		}
	
	    int getOwnIndex() {
	        string name = this.gameObject.name;
	        char cindex = name[name.Length - 1];
	        int result = cindex - '0';
	        return result;
	    }
	
	    //检测进入自己区域的hero
	    void checkNearByHero () {
	        if (gameStatusOp.getHeroStandOnArea() == ownIndex) {    //只有当走进自己的区域
	            if (!isCatching) {
	                isCatching = true;
	                addAction.addDirectMovement(this.gameObject);
	            }
	        }
	        else {
	            if (isCatching) {    //刚才为捕捉状态，但此时hero已经走出所属区域
	                gameStatusOp.heroEscapeAndScore();
	                isCatching = false;
	                addAction.addRandomMovement(this.gameObject, false);
	            }
	        }
	    }
	
	    void OnCollisionStay(Collision e) {
	        //撞击围栏，选择下一个点移动
	        if (e.gameObject.name.Contains("Patrol") || e.gameObject.name.Contains("fence")
	            || e.gameObject.tag.Contains("FenceAround")) {
				Debug.Log ("Pump wall");
	            isCatching = false;
	            addAction.addRandomMovement(this.gameObject, false);
	        }
	
	        //撞击hero，游戏结束
	        if (e.gameObject.name.Contains("hero")) {
	            gameStatusOp.patrolHitHeroAndGameover();
				isCatching = false;
				addAction.addStop ();
	        }
	    }
	}
	```
- 还有其他一些联系各个脚本动作的部分代码，以及界面和用户交互代码就不赘述了，见[Github](https://github.com/ZhengweiZhao/Unity3d-hw7)
## 游戏界面效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026212211361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDM3NzY5MQ==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026212247673.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDM3NzY5MQ==,size_16,color_FFFFFF,t_70)

- 最后

[视频链接1🔗](https://pan.baidu.com/s/1nATWmaCDgmYiGiuYL5oZig)

[视频链接2🔗](https://pan.baidu.com/s/133gLFzjZiCauSo3Ev-WcTw)

[我的Github代码传送门](https://github.com/ZhengweiZhao/Unity3d-hw7)

