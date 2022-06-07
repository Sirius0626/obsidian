cmd指令：‘$start_debug_ai'
```python
@gm_cmd("$start_debug_ai", (Bool('是否自动打开行为树')，)，"开启AIDebug", ALL_GAME, classify=GM_GLASSIFY['MENU_COMBAT'])
def start_debug_ai(entity, auto_open_bt):
	entity.start_debug_ai(auto_open_bt)
	return "SUCCESS"
```

`server_ai_comp`的实现逻辑：
每个`tick`调用`ai_debug_tick()`生成`debug_info`，`ai_debug_tick()`遍历`space`中所有的`entity`然后拿取他们的`ai_ctrl`
```python
def start_debug_ai():
	...
	self.add_repeat_timer(0.05, lambda: self.ai_debug_tick(auto_open_bt))


def ai_debug_tick():
	def iter_all_entities():
		for entity_id in self.space.entities:
			yield EntityManager.getentity(entity_id)
		yield self.space
	...
	
	
	for entity in inter_all_entities():
		if not hasattr(entity, "ai_ctrl") or entity.ai_ctrl is None:
			continue
		info = debug_info.setdefault(entity.id, {})
	...
	self.client.ai_debug_info(debug_info,auto_open_bt)
```
`client_ai_comp`
### space
`space_id = genv.player.space.id`
ps: 服务端没有`genv.player`
`space = EntityManager.getentity(space_id)`

### 包体调试
根据`$start_debug_ai`的逻辑，只需要更改用于遍历的`space`为包体的就行，具体操作如下
弄一个新指令`$push_space_id`，可以将当前的`space_id`上传到服务器的`genv.space_id_for_debug_ai`中，然后`$start_debug_ai`中新增一个参数`windows_no_editor`判断是否是包体测试，从而使用`self.space`或是`genv.space_id_for_debug_ai`
```python
@gm_cmd("$push_space_id", (), "上传当前space的id", ALL_GAME, classify=GM_CLASSIFY['MENU_COMBAT'])
def push_space_id(entity):

    entity.push_space_id()

    return "SUCCESS"
# 增加包体测试参数
@gm_cmd("$start_debug_ai", (Bool('是否自动打开行为树'), Bool('是否是包体测试'),), "开启AIDebug", ALL_GAME, classify=GM_CLASSIFY['MENU_COMBAT'])
def start_debug_ai(entity, auto_open_bt, windows_no_editor=False):

    entity.start_debug_ai(auto_open_bt, windows_no_editor)

    return "SUCCESS"
```

```python
def push_space_id(self):
	if not self.space:
		return
	genv.space_id_for_ai_debug = self.space.id

def ai_debug_tick():
	sapce = self.space
	if space_id:
		space = EntityManager.getentity(space_id)
	...
```