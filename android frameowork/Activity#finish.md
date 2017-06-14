# Activity.finish

Activity有2个finish方法。

Activity.finish -> finish(false)
Activity.finishAndRemoveTask -> finish(true)

```
/**
     * Finishes the current activity and specifies whether to remove the task associated with this
     * activity.
     */
    private void finish(boolean finishTask) {
        if (mParent == null) {
            int resultCode;
            Intent resultData;
            synchronized (this) {
                resultCode = mResultCode;
                resultData = mResultData;
            }
            if (false) Log.v(TAG, "Finishing self: token=" + mToken);
            try {
                if (resultData != null) {
                    resultData.prepareToLeaveProcess();
                }
                if (ActivityManagerNative.getDefault()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
            } catch (RemoteException e) {
                // Empty
            }
        } else {
            mParent.finishFromChild(this);
        }
    }
```
