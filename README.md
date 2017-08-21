# build-ros-on-macos
Some notes for a succesful build ROS on MacOS/Sierra


After several failed attempts building ROS/kinetic/release branch, i have landed successfully on the upstream-development.

I have referred from follow sources
    -
    -

# ENVIRONMENT
- MacOS/Sierra (10.12.6).
- Xcode: Xcode 8.3.3 (Build version 8E3004b)
- ROS/Kinetic/desktop_full/upstream-development

# BUILD STEP

### Clean up brew-based pip installed packages
    pip freeze --local | xargs sudo -H pip uninstall -y
    rm -rf ~/Library/Caches/pip
    rm -rf ~/Library/Logs/pip

### Clean up installed packages and remove homebrew completely  
    brew remove --force $(brew list) --ignore-dependencies
    brew prune
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
    sudo rm -rf /usr/local/lib/python*
    rm /usr/local/share/sip/PyQt5
    rmdir ~/Library/Caches/Homebrew
    rmdir ~/Library/Logs/Homebrew

### Clean up previous ROS's running settings
    cd ~
    rm -rf ~/.ros
    rm -rf ~/.rviz

### Clean up previous catkin workspace
    rm -rf ~/ros_catkin_ws
    mkdir ~/ros_catkin_ws
    cd ~/ros_catkin_ws

### Install new homebrew and required taps
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
    brew tap ros/deps
    brew tap osrf/simulation
    brew tap homebrew/science

### Install python (2.7)
    brew install python
    export PATH="/usr/local/opt/python/libexec/bin:$PATH" # or put this line in ~/.bash_profile
    echo 'export PATH="/usr/local/opt/python/libexec/bin:$PATH"' >> ~/.bash_profile
    #mkdir -p ~/Library/Python/2.7/lib/python/site-packages
    #echo "$(brew --prefix)/lib/python2.7/site-packages" >> ~/Library/Python/2.7/lib/python/site-packages/homebrew.pth

### Install Qt5 and PyQt5
    brew install qt # Qt5
    export PATH="/usr/local/opt/qt/bin:$PATH" # or put this line in ~/.bash_profile
    echo 'export PATH="/usr/local/opt/qt/bin:$PATH"' >> ~/.bash_profile
    brew install pyqt # PyQt5
    ln -s /usr/local/share/sip/Qt5 /usr/local/share/sip/PyQt5

### Install dependencies
    brew opencv3 --without-numpy
    export PATH="/usr/local/opt/opencv3/bin:$PATH" # or put this line in ~/.bash_profile
    echo 'export PATH="/usr/local/opt/opencv3/bin:$PATH"' >> ~/.bash_profile

### Install Gazebo (I use gazebo8 instead gazebo7)
    brew install gazebo8

### Install other packages
    # required packages
    brew install gtk+ pcl poco tinyxml2 ossp-uuid eigen bullet urdfdom assimp qhull collada-dom yaml-cpp fltk sdl_image theora
    # optional packages
    brew install openni openni2 librealsense wxmac wxpython

### Install build helper packages
    sudo -H pip install -U wstool rosdep rosinstall rosinstall_generator rospkg catkin_tools catkin_pkg empy nose sphinx distribute bloom

###
    sudo rm /etc/ros/rosdep/sources.list.d/20-default.list
    sudo rosdep init --verbose
    rosdep update --verbose

### Generate .rosinstall file for desktop_full and relating packages
    #rosinstall_generator desktop_full --rosdistro kinetic --deps --wet-only --tar --verbose --upstream-development > kinetic-desktop_full-wet.rosinstall
    rosinstall_generator \
        desktop_full \
        control_toolbox realtime_tools ros_control navigation map_msgs perception_pcl bfl \
        --rosdistro kinetic --deps --wet-only --tar --upstream-development > kinetic-desktop_full-wet.rosinstall

### Initiate catkin workspace
    wstool init -j8 src kinetic-desktop_full-wet.rosinstall

### Modify
    wstool set stage_ros --git https://github.com/ros-simulation/stage_ros.git --version=indigo-devel -t src -y

###
    #roslocate info control_toolbox --distro kinetic > kinetic-desktop_full-wet-control_toolbox.rosinstall
    #roslocate info realtime_tools --distro kinetic > kinetic-desktop_full-wet-realtime_tools.rosinstall
    #roslocate info ros_control --distro kinetic > kinetic-desktop_full-wet-ros_control.rosinstall
    #roslocate info navigation --distro kinetic > kinetic-desktop_full-wet-navigation.rosinstall
    #roslocate info bfl --distro kinetic > kinetic-desktop_full-wet-orocos-bfl.rosinstall
    #roslocate info map_msgs --distro kinetic > kinetic-desktop_full-wet-map_msgs.rosinstall
    #roslocate info perception_pcl --distro kinetic > kinetic-desktop_full-wet-perception_pcl.rosinstall

    #wstool merge -t src kinetic-desktop_full-wet-control_toolbox.rosinstall -y
    #wstool merge -t src kinetic-desktop_full-wet-realtime_tools.rosinstall -y
    #wstool merge -t src kinetic-desktop_full-wet-ros_control.rosinstall -y
    #wstool merge -t src kinetic-desktop_full-wet-navigation.rosinstall -y
    #wstool merge -t src kinetic-desktop_full-wet-orocos-bfl.rosinstall -y
    #wstool merge -t src kinetic-desktop_full-wet-map_msgs.rosinstall -y
    #wstool merge -t src kinetic-desktop_full-wet-perception_pcl.rosinstall -y

### Update sources
    wstool update -j8 -t src

### Check and install ROS's requirements
    rosdep install --from-paths src --ignore-src --rosdistro kinetic -y --skip-keys="opencv3"

### Upgrade all install python’s packages
    sudo -H pip install -U $(pip freeze | awk '{split($0, a, "=="); print a[1]}')

### Fix failed build relating to include header
    sed -i '.bak' 's/find_path(UUID_INCLUDE_DIRS uuid\/uuid.h)/find_path(UUID_INCLUDE_DIRS ossp\/uuid.h)/g' src/cmake_modules/cmake/Modules/FindUUID.cmake

### Fix failed build image_publisher relating to opencv
### by message like "Undefined symbols for architecture x86_64:  "cv::VideoCapture::…” due to the short of linking to opencv lib
### MODIFY src/image_pipeline/image_publisher/CMakeLists.txt:
    sed -i '.bak' \
        -e '/catkin_package()/a \
            find_package(OpenCV REQUIRED)' \
        -e 's/target_link_libraries(${PROJECT_NAME} ${catkin_LIBRARIES})/target_link_libraries(${PROJECT_NAME} ${catkin_LIBRARIES} ${OpenCV_LIBRARIES})/g' \
        src/image_pipeline/image_publisher/CMakeLists.txt

### Fix failed build Octovis since QGLViewer not found (not hosted on homebrew), try build QGLViewer from attached source code
    pushd src/octomap/octovis/src/extern/QGLViewer
    qmake -spec macx-g++
    make
    #make install
    popd

### Build octovis with specific options
    src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release -DOCTOVIS_QT5=YES -DCMAKE_CXX_STANDARD=11 --pkg=octovis

### Build rest of packages
    src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release

### Run a master and rviz
    source ~/ros_catkin_ws/install_isolated/setup.bash
    export ROS_HOSTNAME=‘localhost'
    roscore & rviz
