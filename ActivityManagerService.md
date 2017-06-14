# TaskRecord

- ActivityStack stack
- ArrayList<ActivityRecord> mActivities，按历史顺序排序，存储了所有该task中的activityRecord

# ActivityStack

- ArrayList<TaskRecord> mTaskHistory:  The back history of all previous (and possibly still running) activities.  It contains #TaskRecord objects.

## 几个重要方法

### topRunningActivityLocked（IBinder token, int taskId）

返回该栈中的，最近的活跃的ActivityRecord，但是不包含token的activityRecord，和taskId的TaskRecord