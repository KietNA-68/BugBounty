![](https://i.imgur.com/OyTrQid.png)

# **Reconnaissance**
Bật Wappalyzer lên thì mình thấy target là 1 website chạy wordpress version 5.9.3
Tiếp theo mình vào wpscan để check version và các CVE liên quan![](https://hackmd.io/_uploads/H1qummju2.png)
Thì thấy cũng khá là nhiều CVE, thôi tạm note để đó =)), đi recon tiếp. Sau khi rà 1 vòng website và vào tab "Site Map" trên burpsuite để xem thử cấu trúc của thằng này
![](https://hackmd.io/_uploads/rkTRmmo_h.png)
Các thông tin cho mình biết: 
+ Website sử dụng các plugin: "google-analytics-for-wordpress,s2member,s2member-secure-file-browser,tensillica-logs,tensillica-screening,usp-pro" trong đó có 2 plugin tensillica là tự dev
+ Website sử dụng theme "tensillica" cũng là 1 theme tự dev


Mình thử search CVE của các plugin s2member và usp-pro thì kết quả:
+ Plugin s2member: chỉ có 1 CVE trong quá khứ `CVE-2011-5082` CVE này đã quá cũ so với hiện tại nên mình bỏ qua
![](https://hackmd.io/_uploads/S1r0Nmsd2.png)
+ Plugin usp-pro: đối với plugin này mình không tìm được CVE trên internet nên thôi chắc bỏ qua

# **Attack Phase**

### **Reflected XSS**
Bước đầu tiên mình target vào các plugin mà website tự dev, bao gồm 2 plugin: `tensillica-logs` và `tensillica-screening`
Đọc file js của `tensillica-screening` thì thấy có 1 POST request gửi đến `admin-ajax` nhưng ko có source thì ko biết được nó làm gì :))
![](https://hackmd.io/_uploads/rJVV_QoOn.png)
Thử payload fuzzing các kiểu cũng không ra gì
![](https://hackmd.io/_uploads/rJ2Rd7i_n.png)
Check thằng còn lại
![](https://hackmd.io/_uploads/By3bYmjOn.png)
Cũng zị lun hà :((
![](https://hackmd.io/_uploads/H1-SFQsuh.png)

Mình quyết định bỏ target  2 plugins này và quay lại trang chủ view-source để xem có gì hot

![](https://hackmd.io/_uploads/SkkNjmsOh.png)

Thấy ứng dụng có gọi đến plugin `s2member-secure-file-browser` và các action tương ứng của nó, tiếp theo mình search code plugin trên github thì ra được 1 cái source update cuối toàn mấy năm về trước

![](https://hackmd.io/_uploads/BykZcXiu3.png)

Thôi clone về đọc thử xem sao

![](https://hackmd.io/_uploads/rk_v9Qo_h.png)

Đầu tiên mình mở file `psk_s2msfb.class.php` để view tổng quan thử

![](https://hackmd.io/_uploads/rkBIc7o_3.png)

Xem qua hàm `init()` thì có add các action, trong đó mình để ý đến 1 action `wp_ajax_nopriv_` thì đây là 1 action cho phép người dùng không cần đăng nhập gọi ra và sử dụng
Gọi đến function `ajax_get_directory` có nhận vào 2 parameter `nonce` và `dir` ở POST method

![](https://hackmd.io/_uploads/rJB0smiun.png)

tiếp theo là hàm validate parameter `dir`

![](https://hackmd.io/_uploads/SkWGnQju2.png)

Tức là khi parameter `dir` có các kí tự như ".." thì sẽ thông báo lỗi, kết hợp nonce ở view-source và các parameter đọc trong source thì mình có request hoàn chỉnh:

![](https://hackmd.io/_uploads/Bkv8h7sOn.png)

Tiếp theo sẽ nhận vào 1 parameter `token` và xử lý ở hàm `recursive_directory`

![](https://hackmd.io/_uploads/Hy_ppQiuh.png)

![](https://hackmd.io/_uploads/BkX-Amou3.png)


![](https://hackmd.io/_uploads/B1hzC7sd3.png)

Hàm này  check nếu `token` rỗng thì sẽ gắn vào biến `$search_inp_value`

![](https://hackmd.io/_uploads/B16FR7j_h.png)

Thì biến này sẽ được in ra giao diện nhưng đi qua hàm `rel_litaral`

![](https://hackmd.io/_uploads/BJyT0miu3.png)

![](https://hackmd.io/_uploads/S1OA0msd2.png)

Hàm `rel_litaral` thực hiện encode các kí tự 
ví dụ: `hello <world>` => `Hello &lt;world&gt;`

=> Ở đây ko thực hiện xss được

Tiếp theo check trường hợp nếu `$token` không rỗng thì sẽ thực hiện nối luôn chuỗi token không thông qua hàm lọc

![](https://hackmd.io/_uploads/BkjPJEo_h.png)

Từ đó ta có PoC:

![](https://hackmd.io/_uploads/Syg3JNsOn.png)
![](https://hackmd.io/_uploads/SJ0n14s_n.png)

### Shortcode Execution
Wordpress có 1 trick để tìm ra các chức năng và page ẩn trên ứng dụng: `/?p=[1->1000000000000]`, khi truyền đúng id page thì wordpress sẽ trả lại đường dẫn tương ứng, nên mình có code 1 đoạn script nhỏ để bruteforce page

![](https://hackmd.io/_uploads/H1HxMVjOn.png)

Ra được các page như hình, còn nhiều page nữa hơn 20k cái, nhưng chụp nhiêu đây cho gọn

![](https://hackmd.io/_uploads/S1UVfEi_h.png)

Thấy có 1 page `/usp_form/tensilica-upload-files/` nghi ngờ nên access vào thử

![](https://hackmd.io/_uploads/rk0ynEsO3.png)

ra 1 form upload chắc là để up file, Mình có up thử thì được truy cập id post trả về để tìm file nhưng không thấy đâu

![](https://hackmd.io/_uploads/B1c9fVjuh.png)




Nhưng bù lại có thể chèn mã html thông qua param `usp-content`

![](https://hackmd.io/_uploads/HyZ8DNs_3.png)
![](https://hackmd.io/_uploads/S1m7h4iu3.png)


Kết quả:

![](https://hackmd.io/_uploads/BJqV24s_3.png)

Có thể khai thác HTML Injection rồi report bug luôn nhưng impact không cao, mình quyết định làm tiếp

Mình nhận ra chức năng này sử dụng plugin `usp-pro` thì có lấy source code trên github về thử xem như nào

![](https://hackmd.io/_uploads/BJvg4Es_n.png)

![](https://hackmd.io/_uploads/B1rGNVsu2.png)

sau đó mình view thử qua file usp-shortcodes.php

![](https://hackmd.io/_uploads/B1l7VNid3.png)

mình chú ý đến line cuối `361` có gọi qua hàm `add_shortcode` của wordpress và truyền thẳng parameter `usp_content` vào. Đối chiếu với CVE của wordpress khi nãy mình vừa recon

![](https://hackmd.io/_uploads/Bk8ZrVi_n.png)

=> CVE ảnh hưởng ở version 5.9.3 tức là hoàn toàn có khả năng để khai thác

tiếp theo mình search các shortcode của plugin `usp-pro` thì ra đường dẫn `https://plugin-planet.com/usp-pro-shortcodes/` chứa thông tin toàn bộ shortcode của plugin

![](https://hackmd.io/_uploads/rJJhBEsu2.png)

Từ đó mình thử chèn 1 shortcode như [usp_files] vào parameter `usp-content`

![](https://hackmd.io/_uploads/S1xCr4odn.png)

Kết quả:

![](https://hackmd.io/_uploads/HktxLNodn.png)

Lên được 1 form upload File => có thể abuse các shortcode để chèn mã độc hại
Tiếp theo mình chọn `[usp_agree]` shortcode bởi vì nó có `script` là thuộc tính để thực thi mã javascript

![](https://hackmd.io/_uploads/ByhjL4i_n.png)

Payload : `[usp_agree script="alert(document.cookie)"]`
Kết quả:

![](https://hackmd.io/_uploads/H1FKhNjdh.png)

![](https://hackmd.io/_uploads/ByPcnVsOn.png)

Report bug xong lấy bounty rồi mới ra RCE =))

![](https://hackmd.io/_uploads/HJro3Eid2.png)
