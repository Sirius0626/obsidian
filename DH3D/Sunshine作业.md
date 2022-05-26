###  GameExample嵌入sunshine
1.获得Sunshine的SDK，然后放入项目文件中，配置相关的工程文件，目的是为了告诉Sunshine新增加的相关游戏的信息。  
project文件的路径在''.\Sunshine\Projects'下，增加project文件在打开sunshine的时候就可以识别项目了，project文件的格式如下
```xml
<?xml version="1.0" encoding="utf-8"?>
<Root>
	<Engine>
		<Name>Python</Name>
		<RootPath>C:\Users\huangrui07\Desktop\Project\GameExample</RootPath>
		<ResPath></ResPath>
		<ScriptPath></ScriptPath>
		<BinPath></BinPath>
		<DllBootArgs></DllBootArgs>
        <RemoteBootArgs>py -3 C:\Users\huangrui07\Desktop\Project\PyChakura\py_chakura.py -m C:\Users\huangrui07\Desktop\Project\GameExample\launch.py -p "-a %s:%s" --window_name Customize</RemoteBootArgs>
        <WindowTitle>Customize</WindowTitle>
	</Engine>
	<Template>
		<Launcher>Lightning</Launcher>
		<Plugin>Lightning,Rainbow,Atmosphere,Galaxy,Teldrassil</Plugin>
		<Args>EditorMode</Args>
	</Template>
	<Plugin>
        <Lightning>
            <HunterProject>Sunshine</HunterProject>
            <HunterConnectCommand>$remote_debug %s %s</HunterConnectCommand>
        </Lightning>
    </Plugin>
</Root>

```
字段说明：需要注意的是

header 1 | header 2
---|---
RootPath | 游戏根目录
RemoteBootArgs | 启动游戏的命令行
WindowTitle | 游戏窗口标题，用于游戏嵌入
Template.Plugin | 默认载入的插件列表

2.配置文件弄好之后，然后就是在代码中实现连接sunshine，在game_start中去初始化SunshineClient实例并且初始化各个需要插件并且注册到SunshineClient中去。
```python
from SunshineSDK.SunshineClient import SunshineClient
from Plugin.TeldrassilPlugin import TeldrassilPlugin
from Plugin.RainbowPlugin import RainbowPlugin

def game_start(self) -> None:
        """
        游戏开始回调
        Returns:
            None
        """
        #注册 sunshine和相应插件
        self.editor_client = SunshineClient()
        self.editor_client.SetGameEncoding("utf-8")
        teldrassil_plugin = TeldrassilPlugin()
        rainbow_plugin = RainbowPlugin()
        self.editor_client.RegisterPlugin(teldrassil_plugin)
        self.editor_client.RegisterPlugin(rainbow_plugin)
        self.editor_client.Connect()
        
        
def update(self):
    if self.editor_client:
        self.editor_client.Update()

```
需要注意定时调用SunshineClient.Update(),要不然会导致sunshine与游戏无法连通

为了完成最基础的功能需求，此次接入两个插件，分别是RainbowPlugin和TeldrassiPlugin
其中RainbowPlugin主要完成两个方法GetAvailableEntities()和GetEntityData()
```python
    from SunshineSDK.Meta import ClassMetaManager
    
    #获取所有entity的key
    def GetAvailableEntities(self):
        return self.entity_manager.get_all_entity_keys()
    
    #获取对应key值Entity的相关数据
    def GetEntityData(self, str):
        entity = self.entity_manager.get_entity_by_key(key)
        
        if not entity:
            return {}
        
        entity_meta = ClassMetaManager.GetClassMeta(entity.__class__.__name__)
        if entity_meta:
            return entity_meta.SerializeData(entity)
        return {} 
```
实现了对应的接口之后，在sunshine界面的左边就会出现当前已有的Entity，选中Entity后sunshine就会返回对应的Entity_key并且传给TeldrassiPlugin插件，sunshine自动打开其对应的行为树界面。  
TelerassiPlugin插件主要负责sunshine的行为树界面，需要实现的接口如下
```python

    # Rainbow会将entity的id传给Teldrassi，通过传来的key识别entity
    def _get_entity(self, uuid):
        return genv.main_game.entity_manager.get_entity_by_key(uuid)
    
    # 获得点击entity的aifile，传回给sunshine（会打开行为树界面）
    def get_ai_name(self, uuid):
        entity = self._getEntity(uuid)
        if not entity:
            return
        self.Context.call_peer("SetAIName", uuid, entity.aifile)
    
    # 点击界面启动按钮，让行为树跑起来    
    def start_ai(self, uuid):
        entity = self._get_entity(uuid)
        if not entity:
            return
        entity.AIExecutor.btree_enable = True
    
```

游戏中其他的主要文件
EntityManager：管理游戏中所有的Entity
```python

class EntityManager():
    def __init__(self):
        #储存的所有的entity的字典
        self.entity_map = {}
        
    #根据key获得对应的Entity
    def get_entity_by_key(key)
    
    #获得所有Entity的key值，Rainbow加载所有Entity的时候需要
    def get_all_entity_keys()
    
    #添加新的Entity
    def add_entity(key, entity)
    
```

Avatar: 具体的Entity相关的属性
```python
 def __init__(aifile=None):
    self.aifile = aifile
    #创建AI的驱动器
    self.AIExecutor = AIExecutor(self, self.aifile)
    
```

AIExecutor：AI驱动器，用于驱动AI文件(sunshine将xml转成的py文件)
```python


def __init__(self, owner, module_name):
    self.btree_module_name = module_name
    self.owner = owner
    
def load_btree_module(self, module_name):
    pass

# 行为树的py文件中运行子节点的方法
def RunnningChileIndex(self, node_name, index=None)
    #index 当前running的节点下标

def execute(self):
    exec_result = bt_consts.FAILURE
    if self.btree_module:
        #tick 一次行为树
        exec_result = self.btree_module.entry(self.owner)
    return exec_result
def update(self):
    self.execute()

```

节点

```python
class move_to_point(CActionNode):
	'''move to the position'''
	x = btnodeproperty('x', float, 'x坐标', 'x坐标', 0.0)
	y = btnodeproperty('y', float, 'y坐标', 'y坐标', 0.0)

	TplCode ='''
		if owner.position[0] == {{ x }} and owner.position[1] == {{ y }}:
			print("you have arrived")
			return bt_consts.SUCCESS
		owner.move_to_point({{ x }}, {{ y }})
		print("im running")
		return bt_consts.RUNNING
		'''  
```
```python
class to_the_enemy(CActionNode):
	'''追击敌人'''
	TplCode = '''
		import genv
		px = abs(owner.position[0] - genv.player.position[0])
		py = abs(owner.position[1] - genv.player.position[1])
		if px == 0 and py == 0:
			print("done")
			return bt_consts.SUCCESS
		if math.sqrt(py * py + (px * px)) < owner.view:
			owner.move_to_point(genv.player.position[0], genv.player.position[1])
			print("im hunting")
			return bt_consts.RUNNING
		return
	'''
```

具体设计游戏节点时遇到的一些问题和思考  
1.设计怪物的AI的时候，常常需要读取player的一些数据（比如坐标），所以player的数据应该作为全局变量方便访问，可以让代码变得简洁一些并且可读性更高 

2.应该为monster创建一个新的avatar模板（monster_avatar)。

3.正式Entity的数据应该储存在什么地方，用什么样的方式读取（读取json数据看似是普遍的解决方案）


