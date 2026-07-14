# c
全国大学生智能车竞赛地瓜机器人智慧医疗挑战赛控制代码
#!/usr/bin/env bash
set -eo pipefail

WORKSPACE="${WORKSPACE:-/userdata/dev_ws}"
CONFIG_SRC="${CONFIG_SRC:-${WORKSPACE}/config/src/origincar}"
SETUP_FILE="${SETUP_FILE:-${WORKSPACE}/install/setup.bash}"
MAP_FILE="${MAP_FILE:-${CONFIG_SRC}/map/race_modify.yaml}"
KEEPOUT_MAP_FILE="${KEEPOUT_MAP_FILE:-${CONFIG_SRC}/map/race_keepout.yaml}"
#航行方向

#WAYPOINTS_FILE="${WAYPOINTS_FILE:-${CONFIG_SRC}/origincar_system/config/waypoints.yaml}"

## ==========新增方向赋值&自动切换航点文件逻辑开始==========
CW_FLAG="/userdata/dev_ws/clockwise.txt"
CCW_FLAG="/userdata/dev_ws/counterclockwise.txt"
DIRECTION="none"

# 读取二维码方向标记文件
if [ -f "${CW_FLAG}" ]; then
    DIRECTION="clockwise"
elif [ -f "${CCW_FLAG}" ]; then
    DIRECTION="counterclockwise"
fi
echo "[start_navigation] Current QR DIRECTION variable: ${DIRECTION}"

# 根据方向强制覆盖WAYPOINTS_FILE，外部传入环境变量优先级最高
if [[ "${DIRECTION}" == "clockwise" ]]; then
    # 顺时针 → waypoints2.yaml
    WAYPOINTS_FILE="${WAYPOINTS_FILE:-${CONFIG_SRC}/origincar_system/config/waypoints2.yaml}"
elif [[ "${DIRECTION}" == "counterclockwise" ]]; then
    # 逆时针 → waypoints.yaml
    WAYPOINTS_FILE="${WAYPOINTS_FILE:-${CONFIG_SRC}/origincar_system/config/waypoints.yaml}"
else
    WAYPOINTS_FILE="${WAYPOINTS_FILE:-${CONFIG_SRC}/origincar_system/config/waypoints.yaml}"
fi
# 外部手动传参WAYPOINTS_FILE > 自动识别文件

echo "[start_navigation] Selected waypoints file: ${WAYPOINTS_FILE}"
## ==========新增方向赋值&自动切换航点文件逻辑结束==========







# 启动等待时间，单位：秒；车载主机启动慢时可以适当加大。
RVIZ_DELAY="${RVIZ_DELAY:-3}"
BRINGUP_DELAY="${BRINGUP_DELAY:-5}"
NAV2_DELAY="${NAV2_DELAY:-30}"
QR_DELAY="${QR_DELAY:-2}"

# 可选功能开关：0=关闭，1=开启。
START_CAMERA="${START_CAMERA:-0}"
START_QR_DETECTOR="${START_QR_DETECTOR:-0}"

# 多点巡航默认关闭，避免运行脚本后小车自动运动。
# 需要跑航点时使用：START_MULTI_POINT=1 bash start_navigation.sh
START_MULTI_POINT="${START_MULTI_POINT:-0}"
WAIT_FOR_NAV_START="${WAIT_FOR_NAV_START:-1}"
NAV_MODE="${NAV_MODE:-path_follower}"
# 是否启动完整 Nav2
# 0 表示只启动地图和 AMCL 定位，适合当前自写平滑巡航
# 1 表示启动完整 Nav2，可用于 RViz 2D Goal Pose 规划导航
START_FULL_NAV2="${START_FULL_NAV2:-0}"
ENABLE_QR_SKIP_FIRST="${ENABLE_QR_SKIP_FIRST:-true}"
QR_TOPIC="${QR_TOPIC:-/qr_info}"
QR_IMAGE_TOPIC="${QR_IMAGE_TOPIC:-/image}"
QR_SCAN_HZ="${QR_SCAN_HZ:-30.0}"
QR_DETECTOR_ENGINE="${QR_DETECTOR_ENGINE:-opencv}"
QR_RESIZE_WIDTH="${QR_RESIZE_WIDTH:-0}"
QR_REPEAT_PUBLISH_INTERVAL="${QR_REPEAT_PUBLISH_INTERVAL:-5.0}"









case "${ENABLE_QR_SKIP_FIRST}" in
  1|true|TRUE|True|yes|YES|Yes|on|ON|On)
    ENABLE_QR_SKIP_FIRST="true"
    ;;
  0|false|FALSE|False|no|NO|No|off|OFF|Off)
    ENABLE_QR_SKIP_FIRST="false"
    ;;
esac


# Load encrypted navigation parameters
ENC_NAV_PARAMS="${ENC_NAV_PARAMS:-${WORKSPACE}/config/scripts/.start_navigation_params.sh.enc}"
NAV_PARAM_KEY="${NAV_PARAM_KEY:-${WORKSPACE}/config/keys/.nav_params.key}"
if [ -f "$ENC_NAV_PARAMS" ] && [ -f "$NAV_PARAM_KEY" ]; then
  source <(openssl enc -aes-256-cbc -pbkdf2 -d -iter 200000 -in "$ENC_NAV_PARAMS" -pass file:"$NAV_PARAM_KEY")
else
  echo "[start_navigation] ERROR: encrypted params or key file missing"
  exit 1
fi
PIDS=()
CLEANED_UP=0

cleanup() {
  if [[ "${CLEANED_UP}" == "1" ]]; then
    return
  fi
  CLEANED_UP=1
  echo
  echo "[start_navigation] Stopping launched processes..."
  for pid in "${PIDS[@]}"; do
    if kill -0 "${pid}" 2>/dev/null; then
      kill "${pid}" 2>/dev/null || true
    fi
  done
  wait 2>/dev/null || true
}

start_process() {
  local name="$1"
  shift
  echo "[start_navigation] Starting ${name}: $*"
  "$@" &
  PIDS+=("$!")
}

if [[ ! -f "${SETUP_FILE}" ]]; then
  echo "[start_navigation] Missing ROS setup file: ${SETUP_FILE}" >&2
  echo "[start_navigation] Build first: cd ${WORKSPACE} && colcon build --symlink-install" >&2
  exit 1
fi

source "${SETUP_FILE}"

# Avoid FastDDS shared-memory lock failures such as
# "Failed init_port fastrtps_port7413: open_and_lock_file failed".
# UDPv4 here is DDS local transport, not the removed external IP controller.
export ROS_DISABLE_LOANED_MESSAGES=1
export RMW_FASTRTPS_USE_QOS_FROM_XML=0
export FASTDDS_BUILTIN_TRANSPORTS=UDPv4
unset FASTRTPS_DEFAULT_PROFILES_FILE

trap cleanup EXIT INT TERM

start_process "RViz2" rviz2
sleep "${RVIZ_DELAY}"

start_process "base bringup" ros2 launch origincar_base origincar_bringup.launch.xml \
  start_camera:="${START_CAMERA}"
sleep "${BRINGUP_DELAY}"


if [[ "${START_QR_DETECTOR}" == "1" ]]; then
  if [[ "${START_CAMERA}" != "1" ]]; then
    echo "[start_navigation] Warning: QR detector needs USB camera image topic, but START_CAMERA is not 1."
  fi
  start_process "QR detector" ros2 run qr_detector qr_detector_node --ros-args \
    -p image_topic:="${QR_IMAGE_TOPIC}" \
    -p qr_topic:="${QR_TOPIC}" \
    -p scan_hz:="${QR_SCAN_HZ}" \
    -p detector_engine:="${QR_DETECTOR_ENGINE}" \
    -p resize_width:="${QR_RESIZE_WIDTH}" \
    -p repeat_publish_interval:="${QR_REPEAT_PUBLISH_INTERVAL}"
  sleep "${QR_DELAY}"
fi

if [[ "${START_FULL_NAV2}" == "1" ]]; then
  start_process "Nav2 AMCL keepout" ros2 launch origincar_system nav2_amcl_keepout.launch.xml \
    map:="${MAP_FILE}" \
    initial_pose_x:="${INITIAL_POSE_X}" \
    initial_pose_y:="${INITIAL_POSE_Y}" \
    initial_pose_a:="${INITIAL_POSE_A}"
else
  start_process "map and AMCL localization" ros2 launch origincar_system map_only.launch.xml \
    map:="${MAP_FILE}" \
    keepout_map:="${KEEPOUT_MAP_FILE}"
fi
sleep "${NAV2_DELAY}"

if [[ "${START_MULTI_POINT}" == "1" ]]; then
  if [[ "${WAIT_FOR_NAV_START}" == "1" ]]; then
    echo
    echo "[start_navigation] RViz2, base bringup, and Nav2 are running."
    echo "[start_navigation] Set the initial pose in RViz2 first."
    read -r -p "[start_navigation] Press Enter here to start multi-point navigation..." _ || true
  fi

  if [[ "${NAV_MODE}" == "nav2_goals" ]]; then
    start_process "multi-point navigation" ros2 launch origincar_system multi_point_nav.launch.xml \
      waypoints_file:="${WAYPOINTS_FILE}" \
      action_type:=ordered_smooth \
      pass_radius:="${PASS_RADIUS}" \
      max_goal_retries:="${MAX_GOAL_RETRIES}"
  else
    start_process "smooth path follower" ros2 launch origincar_system smooth_path_follower.launch.xml \
      waypoints_file:="${WAYPOINTS_FILE}" \
      pass_radius:="${PASS_RADIUS}" \
      lookahead_distance:="${LOOKAHEAD_DISTANCE}" \
      linear_speed:="${LINEAR_SPEED}" \
      channel_linear_speed:="${CHANNEL_LINEAR_SPEED}" \
      channel_waypoint_ranges:="${CHANNEL_WAYPOINT_RANGES}" \
      max_angular_z:="${MAX_ANGULAR_Z}" \
      turn_angular_gain:="${TURN_ANGULAR_GAIN}" \
      turn_min_speed_scale:="${TURN_MIN_SPEED_SCALE}" \
      reverse_angular_gain:="${REVERSE_ANGULAR_GAIN}" \
      reverse_min_speed_scale:="${REVERSE_MIN_SPEED_SCALE}" \
      reverse_goal_lookahead_distance:="${REVERSE_GOAL_LOOKAHEAD_DISTANCE}" \
      reverse_pass_radius:="${REVERSE_PASS_RADIUS}" \
      obstacle_stop_distance:="${OBSTACLE_STOP_DISTANCE}" \
      obstacle_enable_from_waypoint:="${OBSTACLE_ENABLE_FROM_WAYPOINT}" \
      obstacle_slow_distance:="${OBSTACLE_SLOW_DISTANCE}" \
      obstacle_avoid_distance:="${OBSTACLE_AVOID_DISTANCE}" \
      front_angle_deg:="${FRONT_ANGLE_DEG}" \
      backup_trigger_time:="${BACKUP_TRIGGER_TIME}" \
      backup_duration:="${BACKUP_DURATION}" \
      backup_speed:="${BACKUP_SPEED}" \
      backup_angular_z:="${BACKUP_ANGULAR_Z}" \
      backup_stop_distance:="${BACKUP_STOP_DISTANCE}" \
      enable_qr_skip_first:="${ENABLE_QR_SKIP_FIRST}" \
      qr_topic:="${QR_TOPIC}"
  fi
else
