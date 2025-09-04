# 修改的内容
## _render.py
修改class Renderer(threading.Thread):的__init__()
```
class Renderer(threading.Thread):
    """This class is responsible for rendering images in the background."""

    def __init__(
        self,
        viewer: "Viewer",
        client: viser.ClientHandle,
        lock: threading.Lock,
        render_task_state: str,#新参数
    ):
        super().__init__(daemon=True)

        self.viewer = viewer
        self.client = client
        self.lock = lock
        self.render_task_state = render_task_state  # 保存参数
```
重写函数
```
    def _get_img_wh(self, aspect: float) -> Tuple[int, int]:
        return 1920, 927
```
对于run函数的
                self.submit(
                    RenderTask("static", self.viewer.get_camera_state(self.client))
                )如下修改
```
                time.sleep(0.1)
            if not self._render_event.wait(0.2):
                if  self.render_task_state=='static':
                    # print('static')
                    self.submit(
                        RenderTask("static", self.viewer.get_camera_state(self.client))
                    # RenderTask("update", self.viewer.get_camera_state(self.client))
                    )
                elif self.render_task_state=='unpdate':
                    self.submit(
                        RenderTask("update", self.viewer.get_camera_state(self.client))#重要，确保实时交互,点击后可以绘制点
                    )
                else:
                    # print('RenderTask("update", self.viewer.get_camera_state(self.client))#重要，确保实时交互')
                    self.submit(
                        RenderTask("update", self.viewer.get_camera_state(self.client))  # 重要，确保实时交互
                    )                
```
对于run函数的最后返回的图像质量设置成100
```
            self.client.scene.set_background_image(
                img,
                format="jpeg",
                jpeg_quality=100 if task.action in ["static", "update"] else 100,
                depth=depth,
            )
```

## viewer. py
新加position到image_height变量
```
@dataclasses.dataclass
class CameraState(object):
    fov: float
    aspect: float
    c2w: Float32[np.ndarray, "4 4"]
    position: float
    look_at:float
    up_direction:float
    image_width: int
    image_height: int
```

class Viewer(object):新增参数
```
    def __init__(
        self,
        server: viser.ViserServer,
        render_fn: Callable,
        output_dir: Optional[Path] = None,
        mode: Literal["rendering", "training"] = "rendering",
        render_task_state: str = 'static',  # 添加参数
    ):
        # Public states.
        self.render_task_state = render_task_state        
```
注释以下内容
```
        # server.gui.set_panel_label("basic viewer")
        # server.gui.configure_theme(
        #     control_layout="collapsible",
        #     dark_mode=True,
        #     brand_color=(255, 211, 105),
        # )
```
修改以下函数
```
    def _init_rendering_tab(self):
        # Allow subclasses to override for custom rendering table
        self.render_tab_state = RenderTabState()
        self._rendering_tab_handles = {}
        # self._rendering_folder = self.server.gui.add_folder("Rendering")#注释此行

    #对照注释代码
    def _populate_rendering_tab(self):
        # Allow subclasses to override for custom rendering table
        # assert self.render_tab_state is not None, "Render tab state is not initialized"
        # assert self._rendering_folder is not None, "Rendering folder is not initialized"
        # with self._rendering_folder:
        #     viewer_res_slider = self.server.gui.add_slider(
        #         "Viewer Res",
        #         min=64,
        #         max=2048,
        #         step=1,
        #         initial_value=2048,
        #         hint="Maximum resolution of the viewer rendered image.",
        #     )
        #
        #     @viewer_res_slider.on_update
        #     def _(_) -> None:
        #         self.render_tab_state.viewer_res = int(viewer_res_slider.value)
        #         self.rerender(_)
        #
        #     self._rendering_tab_handles["viewer_res_slider"] = viewer_res_slider

        # training tab handles should also be disabled during dumping video.
        extra_handles = self._rendering_tab_handles.copy()
        if self.mode == "training":
            extra_handles.update(self._training_tab_handles)
        # handles = populate_general_render_tab(
        #     self.server,
        #     output_dir=self.output_dir,
        #     folder=self._rendering_folder,
        #     render_tab_state=self.render_tab_state,
        #     extra_handles=extra_handles,
        # )
        # self._rendering_tab_handles.update(handles)

```
在Viewer类，_connect_client函数前，加新函数
```
    def set_render_task_state(self, state: bool):
        """更新并通知所有渲染器"""
        self.render_task_state = state
        for renderer in self._renderers.values():
            renderer.render_task_state = state
```




新加        client.camera.position = (0.0, 0.0, 5.0)
        client.camera.look_at = (0.0, 0.0, 0.0)
        self._renderers[client_id] = Renderer(
            viewer=self, client=client, lock=self.lock,render_task_state=self.render_task_state
        )
```
    def _connect_client(self, client: viser.ClientHandle):
        client.camera.position = (0.0, 0.0, 5.0)  # -20 20 25
        client.camera.look_at = (0.0, 0.0, 0.0)
        client_id = client.client_id
        self._renderers[client_id] = Renderer(
            viewer=self, client=client, lock=self.lock,render_task_state=self.render_task_state
        )
        self._renderers[client_id].start()
```
设置return 返回
```
    def get_camera_state(self, client: viser.ClientHandle) -> CameraState:
        camera = client.camera
        c2w = np.concatenate(
            [
                np.concatenate(
                    [vt.SO3(camera.wxyz).as_matrix(), camera.position[:, None]], 1
                ),
                [[0, 0, 0, 1]],
            ],
            0,
        )
        return CameraState(
            # fov=camera.fov,
            fov=0.1,
            aspect=camera.aspect,
            c2w=c2w,
            position=camera.position,
            look_at=camera.look_at,
            up_direction=camera.up_direction,
            image_height=camera.image_height,
            image_width=camera.image_width,
        )
```

