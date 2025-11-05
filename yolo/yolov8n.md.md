
```python
import cv2
import numpy as np
import time
from ultralytics import YOLO
import mediapipe as mp
from collections import defaultdict

# --------------------------
# 1. 初始化模型和参数
# --------------------------
# 加载YOLOv8模型（人脸检测用预训练模型，香烟/手机需自定义训练）

yolo_face = YOLO("weights/yolov8n-face.pt")  # 人脸检测模型     
yolo_objects = YOLO("weights/yolov8n.pt")    # 通用目标检测     
yolo_driver = YOLO("weights/driver.pt")      # 驾驶员行为分析
yolo_cigarette = YOLO("weights/smoking_detection_trained2.pt") # 抽烟检测

# MediaPipe人脸特征点提取
mp_face_mesh = mp.solutions.face_mesh
face_mesh = mp_face_mesh.FaceMesh(
    static_image_mode=False,
    max_num_faces=1,
    refine_landmarks=True,  # 高精度模式（468点）
    min_detection_confidence=0.5,
    min_tracking_confidence=0.5
)

# 可视化参数
COLOR_FACE = (0, 255, 0)       # 人脸框颜色（绿色）
COLOR_LANDMARKS = (255, 0, 0)  # 特征点颜色（红色）
COLOR_PHONE = (0, 0, 255)      # 手机框颜色（蓝色）
COLOR_CIGARETTE = (255, 255, 0)# 香烟框颜色（黄色）
FONT = cv2.FONT_HERSHEY_SIMPLEX
FONT_SIZE = 0.7
FONT_THICKNESS = 2

# 行为判断阈值
EAR_THRESH = 0.2       # 眼睛闭合阈值（瞌睡）
MOUTH_THRESH = 0.3     # 嘴巴开合阈值（打哈欠）
PHONE_CLASS_ID = 67    # COCO中手机的类别ID
CIGARETTE_CLASS_ID = 0 # 自定义训练的香烟类别ID（需替换）


# --------------------------
# 2. 辅助函数：特征计算与行为判断
# --------------------------
def calculate_ear(landmarks, img_w, img_h):
    """计算眼睛纵横比（EAR），判断是否闭合"""
    # 左眼关键点索引（MediaPipe 468点）
    LEFT_EYE = [362, 385, 387, 263, 373, 380]
    # 右眼关键点索引
    RIGHT_EYE = [33, 160, 158, 133, 153, 144]
    
    def _get_xy(landmark_idx):
        """获取特征点的(x,y)坐标"""
        lm = landmarks[landmark_idx]
        return (int(lm.x * img_w), int(lm.y * img_h))
    
    def _eye_ratio(eye_indices):
        # 提取眼睛6个关键点坐标
        p1, p2, p3, p4, p5, p6 = [_get_xy(i) for i in eye_indices]
        # 垂直距离
        v1 = np.linalg.norm(np.array(p2) - np.array(p6))
        v2 = np.linalg.norm(np.array(p3) - np.array(p5))
        # 水平距离
        h = np.linalg.norm(np.array(p1) - np.array(p4))
        return (v1 + v2) / (2.0 * h) if h != 0 else 0
    
    left_ear = _eye_ratio(LEFT_EYE)
    right_ear = _eye_ratio(RIGHT_EYE)
    return (left_ear + right_ear) / 2.0


def calculate_mouth_ratio(landmarks, img_w, img_h):
    """计算嘴巴开合度（上下唇距离/面部宽度）"""
    # 关键点索引（上唇、下唇、面部两侧）
    UPPER_LIP = 13    # 上唇中点
    LOWER_LIP = 14    # 下唇中点
    LEFT_CHEEK = 234  # 左脸颊
    RIGHT_CHEEK = 454 # 右脸颊
    
    def _get_xy(idx):
        lm = landmarks[idx]
        return (lm.x * img_w, lm.y * img_h)
    
    ux, uy = _get_xy(UPPER_LIP)
    lx, ly = _get_xy(LOWER_LIP)
    left_x, _ = _get_xy(LEFT_CHEEK)
    right_x, _ = _get_xy(RIGHT_CHEEK)
    
    # 嘴巴高度（y方向）和面部宽度（x方向）
    mouth_height = abs(ly - uy)
    face_width = abs(right_x - left_x)
    return mouth_height / face_width if face_width != 0 else 0


def detect_behavior(landmarks, img_w, img_h, phone_boxes, cigarette_boxes):
    """判断当前帧的行为"""
    behaviors = []
    
    # 1. 打哈欠判断
    mouth_ratio = calculate_mouth_ratio(landmarks, img_w, img_h)
    if mouth_ratio > MOUTH_THRESH:
        behaviors.append(f"fatigue (box: {mouth_ratio:.2f})")
    
    # 2. 瞌睡判断（闭眼）
    ear = calculate_ear(landmarks, img_w, img_h)
    if ear < EAR_THRESH:
        behaviors.append(f"sleep (EAR: {ear:.2f})")
    
    # 3. 玩手机判断（检测到手机且低头，简化判断）
    if len(phone_boxes) > 0:
        # 低头判断：鼻尖y坐标 > 眼睛y坐标
        nose_y = landmarks[1].y * img_h  # 鼻尖
        eye_y = landmarks[362].y * img_h # 左眼上缘
        if nose_y > eye_y + 10:  # 阈值可调整
            behaviors.append("phone")
        else:
            behaviors.append("see phone")
    
    # 4. 抽烟判断（检测到香烟且靠近嘴巴）
    if len(cigarette_boxes) > 0:
        # 香烟框中心
        c_x, c_y = cigarette_boxes[0]["center"]
        # 嘴巴中心（上唇+下唇中点）
        mouth_x = (landmarks[13].x + landmarks[14].x) / 2 * img_w
        mouth_y = (landmarks[13].y + landmarks[14].y) / 2 * img_h
        # 距离判断
        distance = np.linalg.norm(np.array([c_x, c_y]) - np.array([mouth_x, mouth_y]))
        if distance < 50:  # 像素距离阈值
            behaviors.append(f"smoke (dis: {int(distance)}px)")
        else:
            behaviors.append(f"has smoke (dis: {int(distance)}px)")
    
    return behaviors if behaviors else ["normal"]


# --------------------------
# 3. 绘制标记函数
# --------------------------
def draw_markers(frame, face_box, landmarks, phone_boxes, cigarette_boxes, behaviors):
    """在帧上绘制所有标记"""
    img_h, img_w = frame.shape[:2]
    marked_frame = frame.copy()
    
    # 1. 绘制人脸框
    if face_box is not None:
        x1, y1, x2, y2 = face_box
        cv2.rectangle(marked_frame, (x1, y1), (x2, y2), COLOR_FACE, 2)
        cv2.putText(marked_frame, "face", (x1, y1-10), FONT, FONT_SIZE, COLOR_FACE, FONT_THICKNESS)
    
    # 2. 绘制人脸特征点（简化：只画眼睛和嘴巴周围的点）
    if landmarks is not None:
        # 筛选关键点位：眼睛（左右各6点）、嘴巴（12点）
        KEY_LANDMARKS = [362, 385, 387, 263, 373, 380,  # 左眼
                         33, 160, 158, 133, 153, 144,   # 右眼
                         61, 185, 40, 39, 37,  # 上唇
                         267, 269, 270, 409,  # 下唇
                         13, 14]  # 嘴唇中点
        for idx in KEY_LANDMARKS:
            x = int(landmarks[idx].x * img_w)
            y = int(landmarks[idx].y * img_h)
            cv2.circle(marked_frame, (x, y), 2, COLOR_LANDMARKS, -1)  # 实心点
    
    # 3. 绘制手机框
    for box in phone_boxes:
        x1, y1, x2, y2 = box["box"]
        cv2.rectangle(marked_frame, (x1, y1), (x2, y2), COLOR_PHONE, 2)
        cv2.putText(marked_frame, "phone", (x1, y1-10), FONT, FONT_SIZE, COLOR_PHONE, FONT_THICKNESS)
    
    # 4. 绘制香烟框
    for box in cigarette_boxes:
        x1, y1, x2, y2 = box["box"]
        cv2.rectangle(marked_frame, (x1, y1), (x2, y2), COLOR_CIGARETTE, 2)
        cv2.putText(marked_frame, "cigarette", (x1, y1-10), FONT, FONT_SIZE, COLOR_CIGARETTE, FONT_THICKNESS)
    
    # 5. 绘制行为标签（右上角）
    for i, behavior in enumerate(behaviors):
        y_pos = 30 + i * 25
        cv2.putText(marked_frame, behavior, (10, y_pos), FONT, FONT_SIZE, (0, 255, 255), FONT_THICKNESS)
    
    return marked_frame


# --------------------------
# 4. 主流程：处理视频并标记
# --------------------------
def process_video(input_path):
    """处理输入视频，生成带标记的输出视频"""
    cap = cv2.VideoCapture(input_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    
    # 初始化视频写入器
    fourcc = cv2.VideoWriter_fourcc(*"mp4v")

    frame_idx = 0
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # --------------------------
        # 检测步骤
        # --------------------------
        # 1. 人脸检测
        face_results = yolo_face(frame, verbose=False)
        face_box = None
        for result in face_results:
            for box in result.boxes:
                x1, y1, x2, y2 = map(int, box.xyxy[0])
                face_box = (x1, y1, x2, y2)
                break  # 只处理第一个人脸
        
        # 2. 人脸特征点提取
        landmarks = None
        if face_box is not None:
            # 裁剪人脸区域加速处理
            x1, y1, x2, y2 = face_box
            face_roi = frame[y1:y2, x1:x2]
            rgb_roi = cv2.cvtColor(face_roi, cv2.COLOR_BGR2RGB)
            mesh_results = face_mesh.process(rgb_roi)
            if mesh_results.multi_face_landmarks:
                # 转换特征点坐标到原图（相对人脸ROI→相对原图）
                face_landmarks = mesh_results.multi_face_landmarks[0].landmark
                for lm in face_landmarks:
                    lm.x = (lm.x * (x2 - x1) + x1) / width  # 归一化到原图宽度
                    lm.y = (lm.y * (y2 - y1) + y1) / height # 归一化到原图高度
                landmarks = face_landmarks
        
        # 3. 目标检测（手机、香烟）
        obj_results = yolo_objects(frame, verbose=False)
        phone_boxes = []
        cigarette_boxes = []
        for result in obj_results:
            for box in result.boxes:
                cls_id = int(box.cls[0])
                x1, y1, x2, y2 = map(int, box.xyxy[0])
                center = ((x1 + x2) // 2, (y1 + y2) // 2)
                if cls_id == PHONE_CLASS_ID:
                    phone_boxes.append({"box": (x1, y1, x2, y2), "center": center})
        
        cigarette_results = yolo_cigarette(frame, verbose=False)
        drinking = []
        cigarette_boxes = []
        for result in cigarette_results:
            # print(result)
            for box in result.boxes:
                # print(result)
                cls_id = int(box.cls[0])
                x1, y1, x2, y2 = map(int, box.xyxy[0])
                center = ((x1 + x2) // 2, (y1 + y2) // 2)
                if cls_id == 0:
                    cigarette_boxes.append({"box": (x1, y1, x2, y2), "center": center})
        

        # 4. 行为判断
        behaviors = ["normal"]
        if landmarks is not None:
            # cigarette_boxes = []
            behaviors = detect_behavior(landmarks, width, height, phone_boxes, cigarette_boxes)
        
        # 5. 绘制标记
        marked_frame = draw_markers(frame, face_box, landmarks, phone_boxes, cigarette_boxes, behaviors)
        
        # 写入输出视频
        # out.write(marked_frame)
        cv2.imshow("检测结果", marked_frame)
        
        
        # 显示进度
        frame_idx += 1
        if frame_idx % 100 == 0:
            print(f"处理进度：{frame_idx}/{total_frames} 帧")

        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    # 释放资源
    cap.release()
    face_mesh.close()
    print(f"标记完成")


# --------------------------
# 5. 运行示例
# --------------------------
if __name__ == "__main__":
    input_video = "video/cigarette.mp4"   # 输入视频路径
    output_video = int(time.time())
    process_video(input_video)
```