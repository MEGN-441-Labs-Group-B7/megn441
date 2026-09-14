# Diagnose /imu/data read hang

> having a problem where the thing is trying to read data forever at /imu/data. what is the issue? do not solve it yet
>
> User context
>
> Attachments:
> - To restore this content, you can run the 'unminimize' command. Expanded Securit…: `~/.codex/attachments/8547f3ca-58d6-4a0d-af6a-fdd3ef44e31d/pasted-text.txt`

<details><summary>4 previous messages</summary>

> I’ll trace where `/imu/data` is read and correlate it with the pasted output. I’ll keep this strictly diagnostic—no code or configuration changes.
>
> <details><summary>Explored a file, a search, a list, ran a command</summary>
>
> <details><summary>Explored a file, a search, a list</summary>
>
> - Read `~/.codex/attachments/8547f3ca-58d6-4a0d-af6a-fdd3ef44e31d/pasted-text.txt`
> - Searched for `/imu/data|imu/data|IMU|Imu` in `.`
> - Listed files
>
> </details>
>
> - Ran `git status --short && find . -maxdepth 2 -type f | sort | sed -n '1,240p'`
>
> </details>
>
> The log already shows a strong lead: the hardware-controller process that should feed the IMU pipeline crashes during startup, while the IMU filter itself stays alive. I’m checking the remappings and the exact failure chain before calling that the root cause.
>
> <details><summary>Explored 10 files, 2 searches, ran 3 commands</summary>
>
> <details><summary>Explored 3 files, a search</summary>
>
> - Read `~/.codex/attachments/8547f3ca-58d6-4a0d-af6a-fdd3ef44e31d/pasted-text.txt`
> - Searched for `imu_filter.launch|bringup.launch|ros_robot_controller|imu_raw|imu_corrected|tf_broadcaster_imu|imu_topic|startup_check` in `ros2_ws`
> - Read `./ros2_ws/src/peripherals/launch/imu_filter.launch.py`
> - Read `./ros2_ws/src/driver/ros_robot_controller/ros_robot_controller/ros_robot_controller_node.py`
>
> </details>
>
> - Ran `find ros2_ws/src -type f \( -name '*bringup*' -o -name '*.launch.py' -o -name '*.yaml' \) | sort`
>
> <details><summary>Explored 7 files</summary>
>
> - Read `./ros2_ws/src/driver/controller/launch/controller.launch.py`
> - Read `./ros2_ws/src/driver/controller/launch/odom_publisher.launch.py`
> - Read `./ros2_ws/src/driver/ros_robot_controller/launch/ros_robot_controller.launch.py`
> - Read `./ros2_ws/src/bringup/launch/bringup.launch.py`
> - Read `./ros2_ws/src/bringup/bringup/startup_check.py`
> - Read `./ros2_ws/src/driver/ros_robot_controller_msgs/CMakeLists.txt`
> - Read `./ros2_ws/src/driver/ros_robot_controller/ros_robot_controller/ros_robot_controller_sdk.py`
>
> </details>
>
> - Ran `git log -n 8 --oneline --decorate --all; git status --short; git ls-files 'ros2_ws/build/**' 'ros2_ws/install/**' | sed -n '1,100p'`
>
> <details><summary>Explored a file, a search</summary>
>
> - Read `~/.codex/attachments/8547f3ca-58d6-4a0d-af6a-fdd3ef44e31d/pasted-text.txt`
> - Searched for `ros2 topic|topic echo|/imu|imu/data|imu_raw|imu_corrected` in `README.md`
>
> </details>
>
> - Ran `git show --stat --oneline cc9e207; git show --format=fuller --no-ext-diff -- ros2_ws/src/driver/ros_robot_controller_msgs/CMakeLists.txt ros2_ws/src/bringup/bringup/startup_check.py ros2_ws/src/peripherals/launch/imu_filter.launch.py | sed -n '1,300p'`
>
> </details>
</details>

The IMU filter is not stuck—it has no upstream data because `ros_robot_controller` crashes.

The fatal error is:

```text
ros_robot_controller_msgs_s__rosidl_typesupport_c...so: file too short
```

That generated ROS message library is truncated/corrupt. The controller dies while creating its `Sbus` publisher at [ros_robot_controller_node.py](./ros2_ws/src/driver/ros_robot_controller/ros_robot_controller/ros_robot_controller_node.py:35), before its publishing thread starts. Therefore:

```text
/ros_robot_controller/imu_raw
        → /imu_corrected
        → Madgwick filter
        → /imu
```

receives no raw IMU messages, so Madgwick repeatedly waits for `imu/data_raw`.

Also, this launch remaps the nominal `/imu/data` output to `/imu` in [imu_filter.launch.py](./ros2_ws/src/peripherals/launch/imu_filter.launch.py:56). Thus `/imu/data` is not the configured final topic anyway.

The undefined `NAMESPACE` and checksum messages are additional problems, but they are not necessary to explain this IMU wait. No changes were made.

> how do we fix?

<details><summary>4 previous messages</summary>

> I’ll turn the diagnosis into a safe repair sequence for the robot, including how to verify the rebuilt type-support library and separate the remaining launch errors. I’ll avoid changing this repository unless a source change is actually required.
>
> <details><summary>Explored 11 files, 3 searches, ran a command</summary>
>
> <details><summary>Explored 7 files, a search</summary>
>
> - Read `./ros2_ws/.zshrc`
> - Read `./ros2_ws/.robotrc`
> - Read `./ros2_ws/.typerc`
> - Searched for `colcon|rosdep|need_compile|setup\.zsh|setup\.bash|build.*ros_robot_controller` in `README.md`
> - Read `./ros2_ws/src/driver/ros_robot_controller_msgs/package.xml`
> - Read `./ros2_ws/src/driver/ros_robot_controller/package.xml`
> - Read `./ros2_ws/src/driver/controller/package.xml`
> - Read `./ros2_ws/src/driver/servo_controller/package.xml`
>
> </details>
>
> - Ran `find ros2_ws/src -maxdepth 3 -name COLCON_IGNORE -o -name AMENT_IGNORE -o -name package.xml | sort | sed -n '1,240p'; du -sh ros2_ws 2>/dev/null`
>
> <details><summary>Explored 4 files, 2 searches</summary>
>
> - Read `./.gitignore`
> - Read `./ros2_ws/src/bringup/package.xml`
> - Read `./ros2_ws/src/peripherals/package.xml`
> - Searched for `checksum|Checksum|CHecksum` in `ros_robot_controller_sdk.py`
> - Read `./ros2_ws/src/driver/ros_robot_controller/ros_robot_controller/ros_robot_controller_sdk.py`
> - Searched for `NAMESPACE\s*=|set_buzzer|set_oled` in `ros2_ws`
>
> </details>
>
> </details>
>
> The repair has two layers. First, regenerate the broken custom-message library; that should restore the IMU pipeline. Then address the separate `startup_check` namespace bug so the full bringup stops reporting an unrelated crash.
>
> <details><summary>Explored 4 files, ran a command</summary>
>
> <details><summary>Explored 4 files</summary>
>
> - Read `./ros2_ws/src/driver/ros_robot_controller/setup.py`
> - Read `./ros2_ws/src/driver/controller/setup.py`
> - Read `./ros2_ws/src/driver/servo_controller/setup.py`
> - Read `./ros2_ws/src/bringup/setup.py`
>
> </details>
>
> - Ran `git status --short`
>
> </details>
</details>

The primary fix is to rebuild the corrupted `ros_robot_controller_msgs` package on the robot. Do not copy `build/` or `install/` from another machine.

Stop the launch with Ctrl-C, then run:

```zsh
cd /home/ubuntu/ros2_ws
source /opt/ros/humble/setup.zsh

# Check that storage and inodes are not full.
df -h .
df -i .

# Preserve only the damaged generated artifacts.
repair_backup="$(mktemp -d /home/ubuntu/ros_robot_controller_msgs-repair.XXXXXX)"
mv build/ros_robot_controller_msgs "$repair_backup/build"
mv install/ros_robot_controller_msgs "$repair_backup/install"
echo "Old artifacts saved at $repair_backup"

# Regenerate the library.
colcon build \
  --packages-select ros_robot_controller_msgs \
  --cmake-args -DBUILD_TESTING=OFF
```

After it reports `1 package finished`, verify the rebuilt library:

```zsh
source /home/ubuntu/ros2_ws/install/setup.zsh

typesupport=/home/ubuntu/ros2_ws/install/ros_robot_controller_msgs/local/lib/python3.10/dist-packages/ros_robot_controller_msgs/ros_robot_controller_msgs_s__rosidl_typesupport_c.cpython-310-aarch64-linux-gnu.so

file -L "$typesupport"

python3 -c "from rosidl_generator_py.import_type_support_impl import import_type_support; import_type_support('ros_robot_controller_msgs'); print('type support: OK')"
```

`file` should report an ARM aarch64 ELF shared object, and Python should print `type support: OK`.

Then restart:

```zsh
ros2 launch bringup bringup.launch.py
```

From another terminal:

```zsh
source /home/ubuntu/ros2_ws/install/setup.zsh
timeout 10s ros2 topic echo --once /ros_robot_controller/imu_raw
timeout 10s ros2 topic echo --once /imu
```

Use `/imu`, not `/imu/data`; the launch file intentionally remaps the filtered output to `/imu`.

A separate source bug will remain in [startup_check.py](./ros2_ws/src/bringup/bringup/startup_check.py:53). Its two topic names should be:

```python
'ros_robot_controller/set_buzzer'
'ros_robot_controller/set_oled'
```

After changing those, rebuild `bringup`. If the rebuilt `.so` becomes “file too short” again, suspect a full or failing eMMC/SD filesystem rather than ROS configuration.
