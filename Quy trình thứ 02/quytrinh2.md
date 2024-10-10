# Quy trình thứ hai

- [Quy trình thứ hai](#quy-trình-thứ-hai)
  - [I. Giới thiệu](#i-giới-thiệu)
  - [II. Triển khai](#ii-triển-khai)
    - [1. Cài đặt công cụ](#1-cài-đặt-công-cụ)
      - [1.1: Trên artifactory](#11-trên-artifactory)
        - [a. Cài đặt `Docker`](#a-cài-đặt-docker)
        - [b. Cài đặt image jfrog](#b-cài-đặt-image-jfrog)
        - [c. Kiểm tra trạng thái của container](#c-kiểm-tra-trạng-thái-của-container)
          - [Sửa lỗi](#sửa-lỗi)
        - [d. Thiết lập jfrog](#d-thiết-lập-jfrog)
        - [e. Thiết lập repository cho dự án](#e-thiết-lập-repository-cho-dự-án)
## I. Giới thiệu
  Trong quy trình thứ nhất, khi chạy pipeline, các bước như run build giống hệt nhau sẽ lại được tái kích hoạt và pipeline sau sẽ xóa toàn bộ kết quả của pipeline trước. Gây tốn tài nguyên cũng như bộ nhớ. Điều này cũng gây ra khó khăn cho việc đối chiếu hiệu năng giữa các phiên bản update, khó rollback lại.

Để giải quyết vấn đề này người ta đưa ra phương pháp
![](images/gitlabCICD.png)
Khi chúng ta build xong, phiên bản sẽ được lưu trữ trên artifactory. Khi cần chạy phiên bản nào ta sẽ kéo phiên bản đó về server để chạy.

Trong quy trình này, ta cũng có `2 cách`:
- Cách thứ nhất mỗi server build và run đều phải cần cài đặt runner

- Cách thứ hai, ở bước deploy chúng ta sẽ sử dụng các công cụ khác như `ansible, puppet hay terraform`. (Các này yêu cầu kiến thức hệ thống tốt và phân quyền đúng đắn)

## II. Triển khai
> Quy trình như sau: 1 server build cài gitlab-runner. Server này sẽ chạy stage build sau đó push code lên Jfrog(artifactory)
> Ở server dev hay server deploy pull artifact và deploy

### 1. Cài đặt công cụ
#### 1.1: Trên artifactory 
##### a. Cài đặt `Docker`
```bash
sudo apt install docker.io -y
```
##### b. Cài đặt image jfrog

```bash
## tạo thư mục làm việc cho jfrog
mkdir -p /tools/jfrog/data
## pull images
docker run --name artifactory-jfrog --restart unless-stopped -v /tools/jfrog/data/:/var/opt/jfrog/artifactory -d -p 8081:8081 -p 8082:8082 releases-docker.jfrog.io/jfrog/artifactory-oss:7.77.5
```

##### c. Kiểm tra trạng thái của container
![](images/checkcontainerjfrog.png)

Ta có thể thấy trạng thái đang là *Restarting(1)*

###### Sửa lỗi
```bash
docker logs artifactory-jfrog
```
Lỗi hiển thị ra
![](images/errorlogJfrog.png)
Đây là lỗi không đủ quyền cho nhóm người dùng 1030
```bash
## cấp quyền cho nhóm người dùng 1030
chown -R 1030:1030 /tools/jfrog
## restart container jfrog
docker restart artifactory-jfrog
```
Trong file hosts của máy Windows: sửa file hosts có nội dung như sau
![](images/windowhostfile.png)

##### d. Thiết lập jfrog
Truy cập vào jfrog bằng port hoặc domain được add trong file hosts
![](images/loginpagejfrog.png)
`User và password` mặc định của web là admin và password
> Xem thông tin tại [Jfrog install by docker ](https://jfrog.com/help/r/jfrog-installation-setup-documentation/install-artifactory-single-node-with-docker)

Reverse proxy bằng nginx
```bash
## download nginx
apt install nginx -y
## thay đổi file cấu hình mặc định của nginx
nano /etc/nginx/sites-available/default
```
Thay đổi thành nội dung bên dưới
![](images/nginxconfig.png)
Tạo file reverse proxy cho jfrog
```bash
nano /etc/nginx/conf.d/jfrog.nxqdevops.tech.conf
```
Nội dung file
```conf
server {
        listen 80;
        server_name jfrog.nxqdevops.tech;
        
        location / {
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
                proxy_pass http://192.168.10.90:8082/;
                proxy_redirect off;
        }
}
```
```bash
systemctl restart nginx
```
Sau khi reverse proxy, chúng ta đã có thể truy cập jfrog mà không cần phải điền port trong đường dẫn
![](images/homepageafterreverse.png)

Sửa đường dẫn trong page jfrog
![](images/configurl.png)
##### e. Thiết lập repository cho dự án
Tại repository tạo repo cho dự án
![](images/configrepo.png)
Chọn generic
![](images/configrepo-2.png)
*Tạo user online-shop và gán quyền cho user vào dự án*
Tạo quyền cho trong phần permission
![](images/createpermissionforrepo.png)
![](images/permforuser.png)
Lưu lại và ta hãy đăng nhập lại bằng tài khoản online-shop
![](images/loginbyonlineshop.png)