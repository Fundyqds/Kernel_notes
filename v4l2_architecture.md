## **v4l2 框架** ##

### **一 相关结构体** ###

*  **v4l2_device**

v4l2_device抽象出了整个输入设备的总结构体, 并不向内核注册设备节点, 主要用于后台逻辑控制.

```
struct v4l2_device {
	struct device *dev;
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_device *mdev;
#endif
	struct list_head subdevs;
	spinlock_t lock;
	char name[V4L2_DEVICE_NAME_SIZE];
	void (*notify)(struct v4l2_subdev *sd,
			unsigned int notification, void *arg);
	struct v4l2_ctrl_handler *ctrl_handler;
	struct v4l2_prio_state prio;
	struct kref ref;
	void (*release)(struct v4l2_device *v4l2_dev);
};
```
dev: 指向所属设备.
mdev: 指向struct media_device.
subdevs: 所有注册的子设备都挂载该链表上.
lock: 自旋锁, 用于同步.
name: 设备名称.
notify: 回调函数, 用于接收子设备的事件通知.
ctrl_handler: 
prio: 设备的优先级别状态.
ref: 引用计数.
release: 释放函数, 当引用计数为0时调用.

* **video_device**

```
struct video_device
{
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
	struct media_intf_devnode *intf_devnode;
	struct media_pipeline pipe;
#endif
	const struct v4l2_file_operations *fops;

	u32 device_caps;

	/* sysfs */
	struct device dev;
	struct cdev *cdev;

	struct v4l2_device *v4l2_dev;
	struct device *dev_parent;

	struct v4l2_ctrl_handler *ctrl_handler;

	struct vb2_queue *queue;

	struct v4l2_prio_state *prio;

	/* device info */
	char name[32];
	int vfl_type;
	int vfl_dir;
	int minor;
	u16 num;
	unsigned long flags;
	int index;

	/* V4L2 file handles */
	spinlock_t		fh_lock;
	struct list_head	fh_list;

	int dev_debug;

	v4l2_std_id tvnorms;

	/* callbacks */
	void (*release)(struct video_device *vdev);
	const struct v4l2_ioctl_ops *ioctl_ops;
	DECLARE_BITMAP(valid_ioctls, BASE_VIDIOC_PRIVATE);

	DECLARE_BITMAP(disable_locking, BASE_VIDIOC_PRIVATE);
	struct mutex *lock;
};
```

entity: media entity.
intf_devnode: 
pipe: 
fops: 指向video deivce的 v4l2_file_operations.
device_caps: 
dev: 
v4l2_dev: 指向所属的v4l2_device结构.
dev_parent: 设备的父节点, 默认指向v4l2_device所属的设备.
ctrl_handler: 
queue:
prio:
name: video device 名称.
vfl_type: v4l2 设备类型.
vfl_dir: 
minor: 次设备号.
num: 
flags:
index:
fh_lock: 自旋锁, 用于fh_list的同步.
fh_list: 所有的v4l2_fh实例都挂在该链表上.
dev_debug:
tvnorms:
release:
ioctl_ops:

* **media_device**

```
struct media_device {
	/* dev->driver_data points to this struct. */
	struct device *dev;
	struct media_devnode *devnode;

	char model[32];
	char driver_name[32];
	char serial[40];
	char bus_info[32];
	u32 hw_revision;

	u64 topology_version;

	u32 id;
	struct ida entity_internal_idx;
	int entity_internal_idx_max;

	struct list_head entities;
	struct list_head interfaces;
	struct list_head pads;
	struct list_head links;

	/* notify callback list invoked when a new entity is registered */
	struct list_head entity_notify;

	/* Serializes graph operations. */
	struct mutex graph_mutex;
	struct media_graph pm_count_walk;

	void *source_priv;
	int (*enable_source)(struct media_entity *entity,
			     struct media_pipeline *pipe);
	void (*disable_source)(struct media_entity *entity);

	const struct media_device_ops *ops;
};
```
dev: 指向所属的设备.
devnode: 


* **v4l2_subdev**

```
struct v4l2_subdev {
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
#endif
	struct list_head list;
	struct module *owner;
	bool owner_v4l2_dev;
	u32 flags;
	struct v4l2_device *v4l2_dev;
	const struct v4l2_subdev_ops *ops;
	const struct v4l2_subdev_internal_ops *internal_ops;
	struct v4l2_ctrl_handler *ctrl_handler;
	char name[V4L2_SUBDEV_NAME_SIZE];
	u32 grp_id;
	void *dev_priv;
	void *host_priv;
	struct video_device *devnode;
	struct device *dev;
	struct fwnode_handle *fwnode;
	struct list_head async_list;
	struct v4l2_async_subdev *asd;
	struct v4l2_async_notifier *notifier;
	struct v4l2_subdev_platform_data *pdata;
}
```

* **media_entity**

```
struct media_entity {
	struct media_gobj graph_obj;	/* must be first field in struct */
	const char *name;
	enum media_entity_type obj_type;
	u32 function;
	unsigned long flags;

	u16 num_pads;
	u16 num_links;
	u16 num_backlinks;
	int internal_idx;

	struct media_pad *pads;
	struct list_head links;

	const struct media_entity_operations *ops;

	int stream_count;
	int use_count;

	struct media_pipeline *pipe;

	union {
		struct {
			u32 major;
			u32 minor;
		} dev;
	} info;
};
```

* **media_interface**

```
struct media_interface {
	struct media_gobj		graph_obj;
	struct list_head		links;
	u32				type;
	u32				flags;
};
```

* **v4l2_fh**

```
struct v4l2_fh {
	struct list_head	list;
	struct video_device	*vdev;
	struct v4l2_ctrl_handler *ctrl_handler;
	enum v4l2_priority	prio;

	/* Events */
	wait_queue_head_t	wait;
	struct mutex		subscribe_lock;
	struct list_head	subscribed;
	struct list_head	available;
	unsigned int		navailable;
	u32			sequence;

#if IS_ENABLED(CONFIG_V4L2_MEM2MEM_DEV)
	struct v4l2_m2m_ctx	*m2m_ctx;
#endif
};
```
list: 链表头, 用于挂载到vdev的fh_list上. 
vdev: 所属的video_device设备.
ctrl_handler: 
prio: 
wait: 等待队列head.
subscribe_lock:
subscribed:
available:
navailable:
sequence:
m2m_ctx:

```
struct v4l2_subscribed_event {
        struct list_head        list;
        u32                     type;
        u32                     id;
        u32                     flags;
        struct v4l2_fh          *fh;
        struct list_head        node;
        const struct v4l2_subscribed_event_ops *ops;
        unsigned int            elems;
        unsigned int            first;
        unsigned int            in_use;
        struct v4l2_kevent      events[];
};
```

```
struct v4l2_ioctl_info {
        unsigned int ioctl;  // ioctl id
        u32 flags;		// 
        const char * const name;
        union {
                u32 offset;
                int (*func)(const struct v4l2_ioctl_ops *ops,
                                struct file *file, void *fh, void *p);
        } u;
        void (*debug)(const void *arg, bool write_only);
};
```