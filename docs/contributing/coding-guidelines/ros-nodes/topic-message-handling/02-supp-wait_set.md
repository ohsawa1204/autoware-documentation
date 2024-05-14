# [supplement] Use rclcpp::WaitSet

## What is `rclcpp::WaitSet`

As explained in [_call take() method of Subscription object_](./index.md#call-take-method-of-subscription-object), `take()` method is irreversible. Once `take()` method is executed, a state of a subscription object changes. Because there is no undo operation against `take()` method, the subscription object can not be restored to its previous state. You can use `rclcpp::WaitSet` before calling `take()` to check the arrival of an incoming message in the subscription queue.
The following sample code shows how `wait_set_.wait()` tells you that a message has already been received and can be obtained by `take()`.

```c++
      auto wait_result = wait_set_.wait(std::chrono::milliseconds(0));
      if (wait_result.kind() == rclcpp::WaitResultKind::Ready &&
          wait_result.get_wait_set().get_rcl_wait_set().subscriptions[0]) {
            sub_->take(msg, msg_info);
            RCLCPP_INFO(this->get_logger(), "Catch message");
            RCLCPP_INFO(this->get_logger(), "I heard: [%s]", msg.data.c_str());
```

A single `rclcpp::WaitSet` object is able to observe multiple subscription objects. If there are multiple subscriptions for different topics, you can check arrival of incoming messages per subscription. Algorithms used in the field of autonomous robot requires multiple incoming messages, like sensor data or actuation state. Using `rclcpp::WaitSet` for the multiple subscriptions, they are able to check whether of not required messages have arrived without taking any message.

```c++
      auto wait_result = wait_set_.wait(std::chrono::milliseconds(0));
      bool received_all_messages = false;
      if (wait_result.kind() == rclcpp::WaitResultKind::Ready) {
        for (auto wait_set_subs : wait_result.get_wait_set().get_rcl_wait_set().subscriptions) {
          if (!wait_set_subs) {
            RCLCPP_INFO_THROTTLE(get_logger(), clock, 5000, "Waiting for data...");
            return {};
          }
        }
        received_all_mesages = true;
      }
```

In the code above, unless `rclcpp::WaitSet` is used, it is impossible to verify arrival of all needed messages without changing state of the subscription objects.

## Coding manner

This section explains the coding manner using `rclcpp::WaitSet` with a sample code below.

- [ros2_subscription_examples/waitset_examples/src/talker_triple.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/talker_triple.cpp)
  - it publishes `/chatter` per one second, `/slower_chatter` per two seconds, and `/slowest_chatter` per three seconds periodically
- [ros2_subscription_examples/waitset_examples/src/timer_listener_triple_async.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/timer_listener_triple_async.cpp)
  - it queries `WaitSet` per one second and if there is a message available, it obtains the message by `take()`
  - it has each subscription for `/chatter` `/slower_chatter`, and `/slower_chatter`

Following three steps are needed to use `WaitSet`.

### 1. Declare and initialize `WaitSet`

You need to declare WaitSet variable first to use.
Below is excerption from [ros2_subscription_examples/waitset_examples/src/timer_listener_triple_async.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/timer_listener_triple_async.cpp).

```c++
rclcpp::WaitSet wait_set_;
```

The `rclcpp::WaitSet` object can be configured during runtime. It is not thread-safe as explained in[_API specification of `rclcpp::WaitSet`_](https://docs.ros.org/en/ros2_packages/humble/api/rclcpp/generated/typedef_namespacerclcpp_1ad6fb19c154de27e92430309d2da25ac3.html)
Thread-safe `WaitSet` are prepared by the `rclcpp` package as listed below.

- [Typedef rclcpp::ThreadSafeWaitSet](https://docs.ros.org/en/ros2_packages/humble/api/rclcpp/generated/typedef_namespacerclcpp_1acaec573e71549fd3078644e18e7f7127.html)
  - Subscription, Timer, and so on can be registered to WaitSet only in thread-safe state
  - sample code is here: [examples/rclcpp/wait_set/src/thread_safe_wait_set.cpp at rolling · ros2/examples](https://github.com/ros2/examples/blob/rolling/rclcpp/wait_set/src/thread_safe_wait_set.cpp)
- [Typedef rclcpp::StaticWaitSet](https://docs.ros.org/en/ros2_packages/humble/api/rclcpp/generated/typedef_namespacerclcpp_1adb06acf4a5723b1445fa6ed4e8f73374.html)
  - Subscription, Timer, and so on can be registered to WaitSet only at initialization
  - here are sample codes:
    - [ros2_subscription_examples/waitset_examples/src/timer_listener_twin_static.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/timer_listener_twin_static.cpp)
    - [examples/rclcpp/wait_set/src/static_wait_set.cpp at rolling · ros2/examples](https://github.com/ros2/examples/blob/rolling/rclcpp/wait_set/src/static_wait_set.cpp)

### 2. Register trigger (Subscription, Timer, and so on) to `WaitSet`

You need to register a trigger to `WaitSet`.
Below is excerption from [ros2_subscription_examples/waitset_examples/src/timer_listener_triple_async.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/timer_listener_triple_async.cpp)

```c++
    subscriptions_array_[0] = create_subscription<std_msgs::msg::String>("chatter", qos, not_executed_callback, subscription_options);
    subscriptions_array_[1] = create_subscription<std_msgs::msg::String>("slower_chatter", qos, not_executed_callback, subscription_options);
    subscriptions_array_[2] = create_subscription<std_msgs::msg::String>("slowest_chatter", qos, not_executed_callback, subscription_options);

    // Add subscription to waitset
    for (auto & subscription : subscriptions_array_) {
      wait_set_.add_subscription(subscription);
    }
```

In the code above, created subscriptions are registered to the `wait_set_` object by `add_subscription()` method.
`rclcpp::WaitSet`-based object basically handles objects each of which have a corresponding callback function. Not only `Subscription` based objects, but `Timer`, `Service` or `Action` based objects can be observed by `rclcpp::WaitSet`-based object. A single `WaitSet` object accepts mixture of different types of objects.
A sample code to register Timer trigger is here.

```c++
wait_set_.add_timer(much_slower_timer_);
```

A trigger can be registered at declaration and initialization as [https://github.com/ros2/examples/blob/rolling/rclcpp/wait_set/src/wait_set_topics_and_timer.cpp#L66](https://github.com/ros2/examples/blob/rolling/rclcpp/wait_set/src/wait_set_topics_and_timer.cpp#L66).

### 3. Verify WaitSet result

The data structure of testing result returned from `WaitSet` is nested.
You can find `WaitSet` result by the following 2 steps;

1. Verify if any trigger has been invoked
2. Verify if a specified trigger has been invoked

For step 1., here is a sample code excerpted from [ros2_subscription_examples/waitset_examples/src/timer_listener_triple_async.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/timer_listener_triple_async.cpp).

```c++
      auto wait_result = wait_set_.wait(std::chrono::milliseconds(0));
      if (wait_result.kind() == rclcpp::WaitResultKind::Ready) {
        RCLCPP_INFO(this->get_logger(), "wait_set tells that some subscription is ready");
      } else {
        RCLCPP_INFO(this->get_logger(), "wait_set tells that any subscription is not ready and return");
        return;
      }
```

In the code above, `auto wait_result = wait_set_.wait(std::chrono::milliseconds(0))` tests whether any trigger in `wait_set_` has been invoked. The argument to the `wait()` method is duration for timeout. If it is greater than 0 milliseconds or seconds, this method will wait for message reception until timeout.
If `wait_result.kind() == rclcpp::WaitResultKind::Ready` is `true`, any trigger has been invoked.

For step 2., here is a sample code excerpted from [ros2_subscription_examples/waitset_examples/src/timer_listener_triple_async.cpp at main · takam5f2/ros2_subscription_examples](https://github.com/takam5f2/ros2_subscription_examples/blob/main/waitset_examples/src/timer_listener_triple_async.cpp).

```c++
      for (size_t i = 0; i < subscriptions_num; i++) {
        if (wait_result.get_wait_set().get_rcl_wait_set().subscriptions[i]) {
          std_msgs::msg::String msg;
          rclcpp::MessageInfo msg_info;
          if (subscriptions_array_[i]->take(msg, msg_info)) {
            RCLCPP_INFO(this->get_logger(), "Catch message via subscription[%ld]", i);
            RCLCPP_INFO(this->get_logger(), "I heard: [%s]", msg.data.c_str());
```

In the code above, `wait_result.get_wait_set().get_rcl_wait_set().subscriptions[i]` indicates whether each individual trigger has been invoked or not. The result is stored to `subscriptions` array. The order in `subscriptions` array is the same as the order in which triggers are registered.