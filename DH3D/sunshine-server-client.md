sunshine获取数据流程
sunshine->client->server->client->sunshine
sunshine通过客户端发送请求，从服务端获取数据发送到客户端，再由客户端转发给sunshine

step1.创建一个新的widget，这个widget可以获得需要的数据文件名，并且提供选择
step2.将数据文件名通过sunshine获取到原始数据
step3.按照一定的格式输出并且用vs code打开

`data_viewer_plugin`
```python
	@sunshine_rpc
	def get_track_list() -> list:
		import infos.both.track as track_package
		suportted_suffix_names = ('.py', '.pyc', '.ues')
		
		track_list = []
		for file_name in os.listdir(os.path.dirname(track_package.__path__)):
			if not file_name.endswith(suportted_suufix_names):
				continue
			module_name, _ = os.path.splitext(file_name)
			if module_name == '__init__':
				continue
			track_list.append(module_name)
		return track_list
```
`sunshine_rpc`装饰器说明这个方法能被sunshine调用
```python
def _get_track_name():
	asyncio.ensure_future(self.get_server_data_show_name())

async def get_server_data_show_name():
	try:
		data = await DataViewer.rpc.get_track_list()
	...
```

在游戏客户端中通过`genv.player.XXXXXXX`调用客户端上发请求
```python
@sunshine_rpc
def get_server_data(self, data_name:str):
	data_class, _, data_name = data_name.partition('/')
	if genv.global_mgr.get_is_in_game() and genv.player:
		if data_class == 'track':
			genv.player.client_get_server_track_data(data_name)
		....
```
client.game.entities.avt_comps.avatar_gm_comp
```python
class PlayerAvatarGMComp(object):
	def client_get_server_track_data()
	self.server.get_server_track_data(data_name)
	...
```
server.entities.avt_comps.avatar_gm_comp
```python
@rpc_method(CLIENT_ONLY, Str())
def get_server_track_data(self, track_name):
	data = {}
	track_module = importlib.import_module(f'infos.both.track.{track_name})
	if track_module:
		data = track_module.data
	self.client.reply_get_server_data(track_name, data)
```
服务端获取数据后，通过回调函数`self.client.xxxxxx`发回信息
```python
@rpc_method(CLIENT_STUB,Str(),Dict())
def reply_get_server_data(self, data_name, data):
	if plugin_mgr.AttachPlugin:
		plugin_mgr.AttachPlugin.sunshine_client.getPlugin('Dh3dDataViewerPlugin').output_data_open_with_vscode(data_name,data)
```
其中，`sunshine_client.getPlugin('Dh3dDataViewer')`
在`innerplugins.client_plugins.sunshine.plugins.data_viewer_plugin`
```python
PLUGIN_NAME = 'Dh3dDataViewerPlugin'
```