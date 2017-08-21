# build-ros-on-macos
Notes while trying to build ROS on MacOS

## Environments
- MacOS/Sierra (10.12.6).
- ROS
    Kinetic
    desktop_full
    upstream-development

## Notes
- I have landed on the upstream-development after several failed attempt on the kinetic/release branch.
- I have referred from:
    +
    +

## CLEAN UP

### Clean up pip packages
    pip freeze --local | xargs sudo -H pip uninstall -y
    rm -rf ~/Library/Caches/pip
    rm -rf ~/Library/Logs/pip

### Clean up Homebrew packages
    brew remove --force $(brew list) --ignore-dependencies
    brew prune
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
    sudo rm -rf /usr/local/lib/python*
    rm /usr/local/share/sip/PyQt5
    rmdir ~/Library/Caches/Homebrew
    rmdir ~/Library/Logs/Homebrew

### Clean up ROS
    cd ~
    rm -rf ~/.ros
    rm -rf ~/.rviz

### Clean up catkin workspace
    rm -rf ~/ros_catkin_ws
    mkdir ~/ros_catkin_ws
    cd ~/ros_catkin_ws

## SETUP

### Install Homebrew and Taps
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
    brew tap ros/deps
    brew tap osrf/simulation
    brew tap homebrew/science
    # brew tap dartsim/dart

### Install python (2.7)
    brew install python
    export PATH="/usr/local/opt/python/libexec/bin:$PATH"
    #mkdir -p ~/Library/Python/2.7/lib/python/site-packages
    #echo "$(brew --prefix)/lib/python2.7/site-packages" >> ~/Library/Python/2.7/lib/python/site-packages/homebrew.pth

### Install Qt5 and PyQt5
    brew install qt # Qt5
    export PATH="/usr/local/opt/qt/bin:$PATH”
    brew install pyqt # PyQt5
    ln -s /usr/local/share/sip/Qt5 /usr/local/share/sip/PyQt5

### Install dependencies
    brew opencv3 --without-numpy
    brew install gazebo8 gtk+ pcl poco tinyxml2 ossp-uuid eigen bullet urdfdom assimp qhull collada-dom yaml-cpp fltk sdl_image theora
    brew install openni openni2 librealsense wxmac wxpython # optional

### Install helper packages
    sudo -H pip install -U wstool rosdep rosinstall rosinstall_generator rospkg catkin_tools catkin_pkg empy nose sphinx distribute bloom

###
    sudo rm /etc/ros/rosdep/sources.list.d/20-default.list
    sudo rosdep init --verbose
    rosdep update --verbose

## BUILD

###
    rosinstall_generator desktop_full --rosdistro kinetic --deps --wet-only --tar --verbose --upstream-development > kinetic-desktop_full-wet.rosinstall | tee rosinstall_generator-log.txt
    wstool init -j8 src kinetic-desktop_full-wet.rosinstall

###
    wstool set stage_ros --git https://github.com/ros-simulation/stage_ros.git --version=indigo-devel -t src -y

###
    roslocate info control_toolbox --distro kinetic > kinetic-desktop_full-wet-control_toolbox.rosinstall
    roslocate info realtime_tools --distro kinetic > kinetic-desktop_full-wet-realtime_tools.rosinstall
    roslocate info ros_control --distro kinetic > kinetic-desktop_full-wet-ros_control.rosinstall
    roslocate info navigation --distro kinetic > kinetic-desktop_full-wet-navigation.rosinstall
    roslocate info bfl --distro kinetic > kinetic-desktop_full-wet-orocos-bfl.rosinstall
    roslocate info map_msgs --distro kinetic > kinetic-desktop_full-wet-map_msgs.rosinstall
    roslocate info perception_pcl --distro kinetic > kinetic-desktop_full-wet-perception_pcl.rosinstall

    wstool merge -t src kinetic-desktop_full-wet-control_toolbox.rosinstall -y
    wstool merge -t src kinetic-desktop_full-wet-realtime_tools.rosinstall -y
    wstool merge -t src kinetic-desktop_full-wet-ros_control.rosinstall -y
    wstool merge -t src kinetic-desktop_full-wet-navigation.rosinstall -y
    wstool merge -t src kinetic-desktop_full-wet-orocos-bfl.rosinstall -y
    wstool merge -t src kinetic-desktop_full-wet-map_msgs.rosinstall -y
    wstool merge -t src kinetic-desktop_full-wet-perception_pcl.rosinstall -y

###
    wstool update -j8 -t src --verbose | tee wstool_update-log.txt

###
    rosdep install --from-paths src --ignore-src --rosdistro kinetic -y --verbose --skip-keys="opencv3"

### Update all install python’s packages
    sudo -H pip install -U $(pip freeze | awk '{split($0, a, "=="); print a[1]}')

### Fix failed build relates to header
    sed -i '.bak' 's/find_path(UUID_INCLUDE_DIRS uuid\/uuid.h)/find_path(UUID_INCLUDE_DIRS ossp\/uuid.h)/g' src/cmake_modules/cmake/Modules/FindUUID.cmake

### Fix failed build Octovis since QGLViewer not found (not hosted on homebrew), try build from attached source code
    pushd src/octomap/octovis/src/extern/QGLViewer
    qmake -spec macx-g++
    make
    #make install
    popd

### Fix failed build image_publisher by message like "Undefined symbols for architecture x86_64:  "cv::VideoCapture::…” due to the short of linking to opencv lib
### MODIFY src/image_pipeline/image_publisher/CMakeLists.txt:
    sed -i '.bak' \
        -e '/catkin_package()/a \
            find_package(OpenCV REQUIRED)' \
        -e 's/target_link_libraries(${PROJECT_NAME} ${catkin_LIBRARIES})/target_link_libraries(${PROJECT_NAME} ${catkin_LIBRARIES} ${OpenCV_LIBRARIES})/g' \
        src/image_pipeline/image_publisher/CMakeLists.txt

###
    src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release -DOCTOVIS_QT5=YES -DCMAKE_CXX_STANDARD=11 --pkg=octovis

    src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release

###
source ~/ros_catkin_ws/install_isolated/setup.bash
export ROS_HOSTNAME=‘localhost'
roscore & rviz
