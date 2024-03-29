# YOLOv7 Navigation 
This simulation tests our Object GPS approximation based on YOLO inference output 

## YOLOv7 Output 
From a given YOLO inference on a frame, we need to extract the classifier (object type), probability, and bounding box. Bounding box is represented as a x, y, w, h with respect to the original frame. 
For YOLOv7 (MOBDrone-YOLOv7) we need to alter the YOLOv7 detect implementation to return these values as a tuple. 

https://github.com/WongKinYiu/yolov7/blob/main/detect.py#L119

```Python
 # Write results
                # This is 
                for *xyxy, conf, cls in reversed(det):
                    if save_txt:  # Write to file
                        # This is our XYWH format of bounding box. 
                        xywh = (xyxy2xywh(torch.tensor(xyxy).view(1, 4)) / gn).view(-1).tolist()  # normalized xywh
                        line = (cls, *xywh, conf) if opt.save_conf else (cls, *xywh)  # label format
                        with open(txt_path + '.txt', 'a') as f:
                            f.write(('%g ' * len(line)).rstrip() % line + '\n')
```

At the end of detect.py function, we need to store predictions as a list:
    [[classifier0, probability0, (x0, y0, w0, h0)], .. [classifierN, probabilityN, (xN, yN, wN, hN)]]
This way we can use:
```Python
objs = detect(net, meta, img, thresh=0.3)

for obj in objs:
    classifier, probability, bbox = obj # A single list in the detections list, representing a single prediction
    x, y, w, h = bbox # Deconstruct tuple
    pts.append([x, y])
```

With this, 
The full GPS approximation from a live YOLOv7 inference on a frame can be done as follows:
```Python
# Run an inference on an image
objs = detect(net, meta, img, thresh=0.3)

# list of center points of all objects detected
pts = []

# Add the center point of each object to the list
for obj in objs:
    classifier, probability, bbox = obj
    x, y, w, h = bbox 
    pts.append([x, y])

if len(pts) > 0:
    # Generate a vector from the focal point to the camera plane for each detected object (x,y,z = right, back, down) relative to the UAV
    obj_vectors = np.hstack((pts_undist, np.ones((pts_undist.shape[0], 1))))


    # calculate the transformation matrix to orient points in the camera in a NED reference frame
    # Corrects for roll, pitch, and heading of the UAV
    mat_transform = matrix_rot_y(attitude.roll) @ matrix_rot_x(-attitude.pitch - CAM_MOUNT_ANGLE * pi / 180) @ matrix_rot_z(-(90 + heading) * pi / 180)

    for i, obj in enumerate(objs):
        classifier, probability, bbox = obj
        obj_vec = obj_vectors[i]

        # transform the vector toward the object to be in a North-East-Down reference frame relative to the UAV
        ned_vec = obj_vec @ mat_transform
        # approximate lattitude and longitude by extending the object's vector from the location and altitude of the UAV down to the ground
        obj_lat = location.lat + location.alt * (ned_vec[0] / ned_vec[2]) * DEGREES_PER_METER
        obj_lon = location.lon + location.alt * (ned_vec[1] / ned_vec[2]) * DEGREES_PER_METER

        msg = self.vehicle.message_factory.command_int_encode(255, 1, 0, 31000, 0, 0, probability, 0, 0, 0, int(obj_lat * 1e7), int(obj_lon * 1e7), 0)

        self.vehicle.send_mavlink(msg)

```