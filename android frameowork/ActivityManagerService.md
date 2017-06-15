# TaskRecord

- ActivityStack stack
- ArrayList<ActivityRecord> mActivities，按历史顺序排序，存储了所有该task中的activityRecord

## 几个重要方法

## topRunningActivityLocked（notTop）
在该TaskRecord中找不等于notTop的，最近活跃(ActivityRecord.mFinishing == false)的ActivityRecord


# ActivityStack

- ArrayList<TaskRecord> mTaskHistory:  The back history of all previous (and possibly still running) activities.  It contains #TaskRecord objects.

- mPausingActivity 如果当前正在pauseActivity，则该变量存储了正在pause的activity。其是否为空即知道是否有activityRecord在pause中
- mResumedActivity，存储着最近resume的activity



## 几个重要方法

### topRunningActivityLocked（IBinder token, int taskId）

返回该栈中的，最近的活跃的ActivityRecord，但是不包含token的activityRecord，和taskId的TaskRecord

### resumeTopActivityLocked(ActivityRecord prev)

确保当前top的activity是resume状态，如果不是，其内部会调用toprecord.app.scheduleResumeActivity来resume。

