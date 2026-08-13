diff --git a/drivers/can/can_loopback.c b/drivers/can/can_loopback.c
index 1234567..fedcba9 100644
--- a/drivers/can/can_loopback.c
+++ b/drivers/can/can_loopback.c
@@ -78,7 +78,14 @@ static void receive_frame(const struct device *dev,
         (frame->flags & CAN_FRAME_RTR) != 0 ? ", RTR frame" : "");

-    filter->rx_cb(dev, &frame_tmp, filter->cb_arg);
+    /* Protect against NULL RX callbacks */
+    if (filter->rx_cb != NULL) {
+        filter->rx_cb(dev, &frame_tmp, filter->cb_arg);
+    } else {
+        LOG_ERR("RX callback is NULL for filter");
+        /* No action: silently drop */
+    }
 }

@@ -115,7 +122,12 @@ static int can_loopback_send(const struct device *dev,
 #endif /* !CONFIG_CAN_FD_MODE */

     if (frame->dlc > max_dlc) {
-        LOG_ERR("DLC of %d exceeds maximum (%d)", frame->dlc, max_dic);
+        /* Fix typo: max_dic → max_dlc */
+        LOG_ERR("DLC of %d exceeds maximum (%d)", frame->dlc, max_dlc);
         return -EINVAL;
     }

@@ -131,7 +143,6 @@ static int can_loopback_send(const struct device *dev,
     loopback_frame.cb = callback;
     loopback_frame.cb_arg = user_data;

-    /* Reject NULL TX callbacks (not allowed in Zephyr API) */
     ret = k_msgq_put(&data->tx_msgq, &loopback_frame, timeout);
     if (ret < 0) {
         LOG_DBG("TX queue full (err %d)", ret);
@@ -165,12 +176,22 @@ static int can_loopback_add_rx_filter(const struct device *dev, can_rx_callback_
     LOG_DBG("Setting filter ID: 0x%x, mask: 0x%x", filter->id, filter->mask);

-    if ((filter->flags & ~(CAN_FILTER_IDE)) != 0) {
-        LOG_ERR("unsupported CAN filter flags 0x%02x", filter->flags);
-        return -ENOTSUP;
-    }
+    /* Reject only known-invalid flags */
+    if (filter->flags & CAN_FILTER_RESERVED) {
+        LOG_ERR("Unsupported CAN filter flags 0x%02x", filter->flags);
+        return -ENOTSUP;
+    }
+
+    /* RX callback must not be NULL */
+    if (cb == NULL) {
+        LOG_ERR("RX callback cannot be NULL");
+        return -EINVAL;
+    }

     k_mutex_lock(&data->mtx, K_FOREVER);
     filter_id = get_free_filter(data->filters);
@@ -245,6 +266,11 @@ static int can_loopback_set_mode(const struct device *dev, can_mode_t mode)
         return -ENOTSUP;
     }
 #endif /* CONFIG_CAN_FD_MODE */

+    /* Avoid redundant mode changes */
+    if (data->common.mode == mode) {
+        return 0;
+    }
+
     data->common.mode = mode;

@@ -330,11 +356,17 @@ static int can_loopback_init(const struct device *dev)
     tx_tid = k_thread_create(&data->tx_thread_data, data->tx_thread_stack,
                              K_KERNEL_STACK_SIZEOF(data->tx_thread_stack),
                              tx_thread, (void *)dev, NULL, NULL,
                              CONFIG_CAN_LOOPBACK_TX_THREAD_PRIORITY,
                              0, K_NO_WAIT);
-    if (!tx_tid) {
-        LOG_ERR("ERROR spawning tx thread");
-        return -1;
-    }
+    /*
+     * k_thread_create never returns NULL.
+     * On failure it returns a negative errno.
+     */
+    if ((intptr_t)tx_tid < 0) {
+        LOG_ERR("Failed to create TX thread (err %d)", (int)(intptr_t)tx_tid);
+        return (int)(intptr_t)tx_tid;
+    }

     k_thread_name_set(tx_tid, dev->name);

@@ -350,6 +382,16 @@ static int can_loopback_get_core_clock(const struct device *dev, uint32_t *rate)
     /* Recommended CAN clock from CiA 601-3 */
     *rate = MHZ(80); /* CAN peripheral clock */

+    /*
+     * Loopback uses a fixed clock because it does not interact with
+     * hardware timing sources. Real CAN controllers derive timing
+     * from PLLs and peripheral clocks; loopback simulates timing
+     * constraints without hardware.
+     */
+
     return 0;
 }
