# From Docker to bootable linux image
---------------------

Lets now put the skills outlined in the `README` for creating custom docker images and apply some reverse engineering to walk from a docker image to a bootable linux image.

Assuming you are on ubuntu development platform 

# 1. Extract the docker image filesystem

    docker pull ubuntu:20.04
    ContainerID=$(docker run -d ubuntu:20.04 /bin/true)
    docker export -o ubuntu20.tar ${ContainerID}
    sudo mkdir -p $HOME/ubuntu20/chroot
    sudo sh -c "tar -xf ubuntu20.tar -C $HOME/ubuntu20/chroot"
    rm -f ubuntu20.tar
    
# 2. Mount points

    sudo mount --bind /dev $HOME/ubuntu20/chroot/dev  
    sudo mount --bind /run $HOME/ubuntu20/chroot/run

# 3. Internet access 

We are going to install packages and internet connection is required. You will need to copy these files `stub-resolv.conf` and `resolv.conf` to the chroot environment

    sudo cp /run/systemd/resolve/* $HOME/ubuntu20/chroot/etc

# 4. Access the environment

    sudo chroot $HOME/ubuntu20/chroot
