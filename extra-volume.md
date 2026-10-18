after extra volume run below cmd 

# 1. Resize the partition to use all available space
sudo growpart /dev/nvme0n1 1

# 2. Resize the filesystem to fill the partition
sudo resize2fs /dev/nvme0n1p1
