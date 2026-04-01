review 

1. ea_registry login INT NOT NULL PRIMARY KEY, -- FK → Orders.login
-> sai. với toàn project chỉ quan tâm đến eaCode. đây là hệ thống phân tích hiệu chỉnh eaCode mà

hiện dữ liệu bảng Orders phục vụ cho việc tính các thông tin eaCode (dùng login chẳng qua cho môi trường dev để lấy dữ liệu trade orders thôi) nên việc ứng dụng kết quả vào logic mới sau này bạn không được dùng login phải dùng eaCode.

2. ea_state_log
bạn cũng dùng login. không được phải dùng eaCode

3. tương tự ea_p1_weekly p2_ ... dùng login là không được. ngay kể base lot hay regime của cũng là với eaCode chứ không liên quan đến login.

tôi yêu cầu bạn thêm logic này vào rule skill
- với toàn project chỉ quan tâm đến eaCode không được dùng liên quan đến login trong các bảng mới. 
_ việc login update date lot realtime hay vào lệnh như nào bạn không cần quan tâm. sẽ có bên team khác làm việc này. chỉ cần chịu nhiệm vụ tính lot và cập nhật database 

