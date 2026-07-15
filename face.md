import cv2 as cv
import os
import threading
import ipywidgets as widgets
from IPython.display import display
from Arm_Lib import Arm_Device
import Arm_Lib
Arm = Arm_Device()
joints_0 = [90, 135, 20, 25, 90, 30]
Arm.Arm_serial_servo_write6_array(joints_0, 1000)
class PositionalPID:
    def __init__(self, P, I, D):
        self.Kp = P
        self.Ki = I
        self.Kd = D
        self.SystemOutput = 0.0
        self.ResultValueBack = 0.0
        self.PidOutput = 0.0
        self.PIDErrADD = 0.0
        self.ErrBack = 0.0

    def SetStepSignal(self, StepSignal):
        Err = StepSignal - self.SystemOutput
        KpWork = self.Kp * Err
        KiWork = self.Ki * self.PIDErrADD
        KdWork = self.Kd * (Err - self.ErrBack)
        self.PidOutput = KpWork + KiWork + KdWork
        self.PIDErrADD += Err
        if self.PIDErrADD > 500:
            self.PIDErrADD = 500
        if self.PIDErrADD < -500:
            self.PIDErrADD = -500
        self.ErrBack = Err

    def SetInertiaTime(self, InertiaTime, SampleTime):
        self.SystemOutput = (InertiaTime * self.ResultValueBack +
                             SampleTime * self.PidOutput) / (SampleTime + InertiaTime)
        self.ResultValueBack = self.SystemOutput
class face_follow:

    # 模型 URL（类级常量，不依赖外部变量）
    _PROTOTXT_URL = ("https://raw.githubusercontent.com/opencv/opencv/"
                     "master/samples/dnn/face_detector/deploy.prototxt")
    _CAFFEMODEL_URL = ("https://github.com/opencv/opencv_3rdparty/raw/"
                       "dnn_samples_face_detector_20170830/"
                       "res10_300x300_ssd_iter_140000.caffemodel")

    @staticmethod
    def _get_model_dir():
        """获取模型文件所在目录（Jupyter 兼容，使用当前工作目录）"""
        import os
        return os.getcwd()

    @classmethod
    def _ensure_model(cls, prototxt_path, caffemodel_path):
        """自动下载 DNN 模型文件，网络不通则提示手动下载"""
        import os
        import urllib.request

        missing = []
        for path, url, name in [
            (prototxt_path, cls._PROTOTXT_URL, "deploy.prototxt"),
            (caffemodel_path, cls._CAFFEMODEL_URL,
             "res10_300x300_ssd_iter_140000.caffemodel"),
        ]:
            if os.path.exists(path):
                continue
            print("下载 %s ..." % name)
            try:
                urllib.request.urlretrieve(url, path)
                print("%s 下载完成" % name)
            except Exception as e:
                print("下载 %s 失败: %s" % (name, e))
                missing.append((name, url))

        if missing:
            print("\n" + "=" * 55)
            print("以下模型文件缺失，请手动下载后放到程序目录：")
            for name, url in missing:
                print("  %s\n    %s" % (name, url))
            print("=" * 55)

    def __init__(self):
        import os
        self.img = None
        self.target_servox = 90
        self.target_servoy = 45
        self.Arm = Arm_Lib.Arm_Device()

        # ===== 模型路径 =====
        model_dir = self._get_model_dir()
        prototxt = os.path.join(model_dir, "deploy.prototxt")
        caffemodel = os.path.join(model_dir, "res10_300x300_ssd_iter_140000.caffemodel")

        # ===== 检测模式选择 =====
        self.detect_mode = None   # "dnn" 或 "haar"

        if os.path.exists(prototxt) and os.path.exists(caffemodel):
            # DNN 模式
            self.net = cv.dnn.readNetFromCaffe(prototxt, caffemodel)
            try:
                self.net.setPreferableBackend(cv.dnn.DNN_BACKEND_OPENCV)
                self.net.setPreferableTarget(cv.dnn.DNN_TARGET_OPENCL)
            except Exception:
                pass
            self.detect_mode = "dnn"
            print("DNN 人脸检测模型加载成功")
        else:
            # Haar Cascade 回退：使用当前目录下的本地文件
            cascade_path = os.path.join(os.getcwd(), "haarcascade_frontalface_default.xml")
            self.face_cascade = cv.CascadeClassifier(cascade_path)
            if self.face_cascade.empty():
                raise RuntimeError("无法加载 Haar Cascade: %s" % cascade_path)
            self.detect_mode = "haar"
            print("Haar Cascade 人脸检测就绪: %s" % cascade_path)

        # ===== 检测置信度阈值 =====
        self.confidence_threshold = 0.6      # DNN 正常阈值
        self.low_confidence_threshold = 0.35 # DNN 低阈值回退

        # ===== 边缘检测参数 =====
        self.edge_margin = 15
        self.recovery_gain_x = 50
        self.recovery_gain_y = 40

        # ===== PID 参数 =====
        self.xservo_pid = PositionalPID(0.8, 0.08, 0.2)
        self.yservo_pid = PositionalPID(0.8, 0.08, 0.2)

        # ===== 低通滤波 =====
        self.smooth_x = 160.0
        self.smooth_y = 120.0
        self.alpha = 0.4

        # ===== 死区 / 限幅 =====
        self.dead_zone = 15
        self.max_angle_change = 6
        self.recovery_max_angle_change = 12

        self.last_target_servox = 90
        self.last_target_servoy = 45

        self.face_lost_count = 0
        self.max_lost_frames = 15
        self.last_face_pos = None
        self.face_detected = False

        # 帧跳过
        self.frame_count = 0
        self.detect_interval = 3
        self._cached_result = None

    def _extract_faces_dnn(self, detections, h, w, min_conf, min_size):
        """从 DNN 检测结果中提取人脸框"""
        faces = []
        for i in range(detections.shape[2]):
            confidence = detections[0, 0, i, 2]
            if confidence < min_conf:
                continue
            box = detections[0, 0, i, 3:7] * [w, h, w, h]
            (x1, y1, x2, y2) = box.astype("int")
            x1, y1 = max(0, x1), max(0, y1)
            x2, y2 = min(w, x2), min(h, y2)
            fw, fh = x2 - x1, y2 - y1
            if fw > min_size and fh > min_size:
                faces.append((x1, y1, fw, fh, confidence))
        return faces

    def _detect_face_dnn(self, bgr):
        """DNN SSD 人脸检测"""
        h, w = bgr.shape[:2]
        blob = cv.dnn.blobFromImage(bgr, 1.0, (300, 300),
                                     (104.0, 177.0, 123.0))
        self.net.setInput(blob)
        detections = self.net.forward()

        faces = self._extract_faces_dnn(detections, h, w,
                                         self.confidence_threshold, 15)
        secondary_detect = (len(faces) == 0)
        if secondary_detect:
            faces = self._extract_faces_dnn(detections, h, w,
                                             self.low_confidence_threshold, 10)
        return faces, secondary_detect

    def _detect_face_haar(self, gray):
        """Haar Cascade 人脸检测（多轮降级策略）"""
        faces_raw = self.face_cascade.detectMultiScale(
            gray, scaleFactor=1.05, minNeighbors=5,
            minSize=(30, 30), flags=cv.CASCADE_SCALE_IMAGE)

        if len(faces_raw) == 0:
            # 降级：更宽松的参数
            faces_raw = self.face_cascade.detectMultiScale(
                gray, scaleFactor=1.03, minNeighbors=3,
                minSize=(20, 20), flags=cv.CASCADE_SCALE_IMAGE)

        faces = []
        for (x, y, w, h) in faces_raw:
            confidence = min(w * h / 20000.0, 0.95)  # 估算置信度
            faces.append((x, y, w, h, confidence))
        return faces, False

    def _check_edge_status(self, fx, fy, fw, fh, w, h):
        """检测人脸是否贴近画面边缘（不完整人脸）"""
        return {
            'left':   fx <= self.edge_margin,
            'right':  (fx + fw) >= (w - self.edge_margin),
            'top':    fy <= self.edge_margin,
            'bottom': (fy + fh) >= (h - self.edge_margin),
            'any':    False  # 占位，后续更新
        }

    def detect_face(self, bgr):
        """
        人脸检测（DNN 或 Haar Cascade，取决于 detect_mode）
        返回 (center_x, center_y, w, h, faces, edge_status) 或 None
        """
        h, w = bgr.shape[:2]

        if self.detect_mode == "dnn":
            faces, secondary_detect = self._detect_face_dnn(bgr)
        else:
            gray = cv.cvtColor(bgr, cv.COLOR_BGR2GRAY)
            faces, secondary_detect = self._detect_face_haar(gray)

        if len(faces) == 0:
            self.last_face_pos = None
            return None

        # 按 面积*置信度 排序
        faces.sort(key=lambda f: f[2] * f[3] * f[4], reverse=True)

        # 时序优先：靠近上一帧位置的加权
        if self.last_face_pos is not None and len(faces) > 1:
            lx, ly, lw, lh = self.last_face_pos
            lcx, lcy = lx + lw / 2, ly + lh / 2

            def score(f):
                fx, fy, fw_, fh_, conf = f
                fcx, fcy = fx + fw_ / 2, fy + fh_ / 2
                dist = ((fcx - lcx) ** 2 + (fcy - lcy) ** 2) ** 0.5 + 1
                return fw_ * fh_ * conf / dist

            faces.sort(key=score, reverse=True)

        (fx, fy, fw, fh, conf) = faces[0]
        center_x = fx + fw / 2
        center_y = fy + fh / 2
        self.last_face_pos = (fx, fy, fw, fh)

        # 检测人脸是否不完整（贴边）
        edge_status = self._check_edge_status(fx, fy, fw, fh, w, h)
        edge_status['any'] = any([edge_status['left'], edge_status['right'],
                                   edge_status['top'], edge_status['bottom']])
        edge_status['secondary'] = secondary_detect

        # 返回不含置信度的框
        faces_clean = [(fx, fy, fw, fh) for (fx, fy, fw, fh, _) in faces]
        return center_x, center_y, fw, fh, faces_clean, edge_status

    def follow_function(self, img):
        self.img = img
        h, w = self.img.shape[:2]
        cx, cy = w // 2, h // 2  # 画面中心

        self.frame_count += 1
        if self.frame_count % self.detect_interval == 0:
            result = self.detect_face(self.img)
            self._cached_result = result
        else:
            result = self._cached_result

        if result is not None:
            face_x, face_y, face_w, face_h, faces, edge_status = result
            self.face_detected = True
            self.face_lost_count = 0

            # ===== 绘制人脸框 =====
            for (fx, fy, fw, fh) in faces:
                # 不完整人脸用橙色框，正常用绿色框
                color = (0, 165, 255) if edge_status['any'] else (0, 255, 0)
                thickness = 3 if edge_status['any'] else 2
                cv.rectangle(self.img, (fx, fy), (fx + fw, fy + fh), color, thickness)
                cv.circle(self.img, (int(fx + fw / 2), int(fy + fh / 2)),
                          5, (0, 0, 255), -1)

            # ===== 计算边缘恢复补偿 =====
            recovery_x = 0.0
            recovery_y = 0.0

            if edge_status['left']:
                recovery_x = self.recovery_gain_x   # 左贴边 → 目标偏右 → 摄像头左转
            elif edge_status['right']:
                recovery_x = -self.recovery_gain_x  # 右贴边 → 目标偏左 → 摄像头右转

            if edge_status['top']:
                recovery_y = self.recovery_gain_y   # 上贴边 → 目标偏下 → 摄像头上仰
            elif edge_status['bottom']:
                recovery_y = -self.recovery_gain_y  # 下贴边 → 目标偏上 → 摄像头下俯

            # PID 目标点 = 画面中心 + 恢复补偿偏移
            pid_target_x = cx + recovery_x
            pid_target_y = cy + recovery_y

            # 绘制恢复方向箭头
            if edge_status['any']:
                # 画面中心十字准星
                cv.line(self.img, (cx - 15, cy), (cx + 15, cy), (0, 255, 255), 1)
                cv.line(self.img, (cx, cy - 15), (cx, cy + 15), (0, 255, 255), 1)
                # 从人脸中心指向恢复目标
                pt1 = (int(face_x), int(face_y))
                pt2 = (int(pid_target_x), int(pid_target_y))
                cv.arrowedLine(self.img, pt1, pt2, (0, 255, 255), 2, tipLength=0.3)
                # 文字提示
                cv.putText(self.img, "Partial Face - Recovering", (5, 50),
                           cv.FONT_HERSHEY_SIMPLEX, 0.5, (0, 165, 255), 2)

            self.smooth_x = self.alpha * face_x + (1 - self.alpha) * self.smooth_x
            self.smooth_y = self.alpha * face_y + (1 - self.alpha) * self.smooth_y

            status_text = "Face Locked" if not edge_status['any'] else "Partial Face"
            if self.detect_mode == "haar":
                status_text += " [Haar]"
            status_color = (0, 255, 0) if not edge_status['any'] else (0, 165, 255)
            cv.putText(self.img, status_text, (5, 20),
                       cv.FONT_HERSHEY_SIMPLEX, 0.6, status_color, 2)

            # 边缘检测二级命中提示
            if edge_status.get('secondary'):
                cv.putText(self.img, "Lo-Conf Detect", (5, 75),
                           cv.FONT_HERSHEY_SIMPLEX, 0.45, (0, 200, 255), 1)

            # ===== X 轴 PID（含恢复偏移） =====
            x_error = abs(self.smooth_x - pid_target_x)
            if x_error > self.dead_zone:
                self.xservo_pid.SystemOutput = self.smooth_x
                self.xservo_pid.SetStepSignal(pid_target_x)
                self.xservo_pid.SetInertiaTime(0.01, 0.1)
                target_valuex = int(1500 + self.xservo_pid.SystemOutput)
                target_servox_new = int((target_valuex - 500) / 10)
                delta_x = target_servox_new - self.last_target_servox
                # 不完整人脸时允许更大步长以便快速恢复
                max_step = self.recovery_max_angle_change if edge_status['any'] else self.max_angle_change
                if abs(delta_x) > max_step:
                    delta_x = max_step if delta_x > 0 else -max_step
                self.target_servox = self.last_target_servox + delta_x
                self.target_servox = max(0, min(180, self.target_servox))

            # ===== Y 轴 PID（含恢复偏移） =====
            y_error = abs(self.smooth_y - pid_target_y)
            if y_error > self.dead_zone:
                self.yservo_pid.SystemOutput = self.smooth_y
                self.yservo_pid.SetStepSignal(pid_target_y)
                self.yservo_pid.SetInertiaTime(0.01, 0.1)
                target_valuey = int(1500 + self.yservo_pid.SystemOutput)
                target_servoy_new = int((target_valuey - 500) / 10) - 45
                delta_y = target_servoy_new - self.last_target_servoy
                max_step = self.recovery_max_angle_change if edge_status['any'] else self.max_angle_change
                if abs(delta_y) > max_step:
                    delta_y = max_step if delta_y > 0 else -max_step
                self.target_servoy = self.last_target_servoy + delta_y
def camera():
    global model
    capture = cv.VideoCapture(0)
    capture.set(3, 320)
    capture.set(4, 240)
    capture.set(5, 30)

    while capture.isOpened():
        try:
            _, img = capture.read()
            img = cv.resize(img, (320, 240))

            if model == 'face_follow' and face_tracker is not None:
                img = face_tracker.follow_function(img)

            if model == 'Exit':
                cv.destroyAllWindows()
                capture.release()
                break

            imgbox.value = cv.imencode('.jpg', img,
                                       [cv.IMWRITE_JPEG_QUALITY, 50])[1].tobytes()
        except KeyboardInterrupt:
            capture.release()
devices = []
for i in range(5):
    cap = cv.VideoCapture(i, cv.CAP_DSHOW)
    if cap.isOpened():
        devices.append(i)
        cap.release()
print("可用设备索引:", devices)

display(controls_box)
                self.target_servoy = max(0, min(360, self.target_servoy))

            self.last_target_servox = self.target_servox
            self.last_target_servoy = self.target_servoy

            joints_0 = [self.target_servox, 135, self.target_servoy / 2,
                        self.target_servoy / 2, 90, 30]
            self.Arm.Arm_serial_servo_write6_array(joints_0, 1000)

        else:
            self.face_lost_count += 1
            self.face_detected = False
            cv.putText(self.img, "Face Lost: %d" % self.face_lost_count, (5, 20),
                       cv.FONT_HERSHEY_SIMPLEX, 0.6, (0, 0, 255), 2)

        # ===== 绘制画面边缘警戒线 =====
        m = self.edge_margin
        cv.rectangle(self.img, (m, m), (w - m, h - m), (80, 80, 80), 1)

        return self.img
face_tracker = None   # 懒加载，点击 Face Follow 时才初始化

model = 'General'

# ===== UI =====
button_layout = widgets.Layout(width='200px', height='80px', align_self='center')

face_follow_btn = widgets.Button(description='Face Follow', button_style='success',
                                  layout=button_layout)
follow_cancel = widgets.Button(description='Stop Follow', button_style='danger',
                                layout=button_layout)
exit_button = widgets.Button(description='Exit', button_style='danger',
                              layout=button_layout)

imgbox = widgets.Image(format='jpg', height=320, width=480,
                       layout=widgets.Layout(align_self='auto'))

img_box = widgets.VBox([imgbox], layout=widgets.Layout(align_self='auto'))
Slider_box = widgets.VBox([face_follow_btn, follow_cancel, exit_button],
                          layout=widgets.Layout(align_self='auto'))
controls_box = widgets.HBox([img_box, Slider_box],
                            layout=widgets.Layout(align_self='auto'))
ef face_follow_Callback(value):
    global model, face_tracker
    if face_tracker is None:
        try:
            face_tracker = face_follow()
        except Exception as e:
            print("人脸追踪初始化失败: %s" % e)
            face_tracker = None
            return
    model = 'face_follow'
    print("人脸追踪已启动 (DNN SSD)")


def follow_cancel_Callback(value):
    global model
    model = 'General'
    print("追踪已取消")


def exit_button_Callback(value):
    global model
    model = 'Exit'
    print("程序退出")
face_follow_btn.on_click(face_follow_Callback)
follow_cancel.on_click(follow_cancel_Callback)
exit_button.on_click(exit_button_Callback)
