---
title: "Clustered Index"
date: 2019-09-08T14:08:32+07:00
draft: false
summary: "Clustered Index là gì và tại sao nó có thể tăng tốc độ đọc bản ghi trong cơ sở dữ liệu quan hệ." 
---
## Những điều cần biết
Nếu bạn có một chút kinh nghiệm về lập trình, hiểu được một số khái niệm cơ bản của cơ sở dữ liệu quan hệ (relational database) và SQL (như là index, primary key) thì sẽ dễ đọc hơn.
## Giới thiệu
Trong bài viết này tôi sẽ trình bày hiểu biết của bản thân về clustered index và tại sao nó có thể giúp bạn cải thiện được tốc độ truy vấn dữ liệu khi  sử dụng các  hệ thống  quản lý cơ sở dữ liệu quan hệ  (Relational database  management system). Bài viết bao gồm:
- Ý tưởng chung của clustered index và secondary index.
- Clustered Index trong MySQL.

## Clustered Index
### B-Tree
Index là một phần tối quan trọng của một cơ sở dữ liệu vì nó giúp tăng tốc việc truy vấn thông tin. Để duy trì index của một bảng, người ta thường sử dụng các cấu trúc dữ liệu như hash, B-Tree, LSM-Tree,... Nhưng khi ai đó nói với bạn là bảng của họ có sử dụng clustered index thì nhiều khả năng cấu trúc dữ liệu họ đang sử dụng để lưu index cho bảng là B-Tree. Có thể hiểu B-tree là một cấu trúc dữ liệu dạng cây mà nút lá sẽ lưu thông tin về một bản ghi trong bảng (row / record) và các nút còn lại (internal node) sẽ lưu các index key và các con trỏ đến nút con của nó. B-Tree đánh đổi không gian dữ liệu (một thứ rất rẻ ở thời điểm hiện tại) và độ phức tạp của việc ghi để tăng tốc độ đọc.
![](https://res.cloudinary.com/vido/image/upload/v1567940701/images/clustered-index/Screenshot_from_2019-09-08_15-14-52_b2vt52.png)
Ví dụ về một bảng lưu thông tin của user, dùng B-Tree để đánh index trên cột `user_id`. Hình trên  mô tả cách storage engine tìm được bản ghi chứa thông tin của user với `user_id` là `251`.  Có thể thấy là mỗi internal node lưu thông tin về các khoảng `user_id` và ở mỗi node việc của storage engine là tìm xem khoảng nào chứa giá trị `user_id` truy vấn và cứ thế tìm ra "đường đi" đến nút lá. Nút lá sẽ chứa key mà bạn đang tìm (đương nhiên rồi) và tham chiếu (reference) đến nơi lưu bản ghi hoặc chính bản ghi. 

Một điều mà bạn cần lưu ý là các node trong key nằm gọn trong 1 page . Có thể hiểu đơn giản page là đơn vị bé nhất để lưu trữ data trong bộ nhớ và đọc dữ liệu từ 1 page là nhanh nhất. Tưởng tượng page là 1 trang sách (get it :laughing:), data mà bạn cần tìm là đoạn văn nằm trong trang sách đó và 2 đoạn văn trên và dưới có thể nói về 2 vấn đề hoàn toàn khác nhau, thậm chí các đoạn văn liên quan đến nhau còn nằm không theo thứ tự và rải rác ở rất nhiều trang (thế giới OS nó là vậy đó, chấp nhận đi). Thời gian bạn đọc hết nội dung trong trang đó sẽ ngắn hơn là thời gian bạn đi tìm một trang nào đó trong mục lục rồi lần mò đến trang đó và đọc. Bằng cách lưu hết thông tin trong 1 page, việc tìm kiếm trong B-Tree trở lên nhanh hơn rất nhiều. 

Hành trình tìm dữ liệu của bạn thực ra là nhảy từ page này sang page khác, ở mỗi page chỉ cần bạn biết địa chỉ là gì (`user_id`) thì bạn sẽ biết phải đi theo đường nào (reference).


Độ phức tạp của B-Tree là O(log N) cho ghi và đọc. Tồn tại qua gần nửa thế kỉ, nó vẫn đang là cấu trúc dữ liệu phổ biến nhất dùng cho cả cơ sở dữ liệu quan hệ và phi quan hệ.

### Clustered Index
Như đã nói thì nút lá có thể lưu tham chiếu đến nơi lưu trữ bản ghi (gọi là heap file) hoặc là lưu luôn bản ghi. Cả 2 phương pháp đều có mặt lợi và hại, nếu phương pháp thứ nhất tránh việc data bị lặp lại (duplicate) ở nhiều nơi trong disk qua đó tích kiệm không gian lưu trữ thì phương pháp thứ 2 sẽ tăng tốc độ đọc (như tôi đã nói thì việc đọc thông tin trong 1 page là nhanh nhất và thời gian tìm đúng page để đọc là rất dài). 
Nếu storage engine của hệ thống cơ sở dữ liệu bạn đang dùng chọn phương pháp thứ 2 cho B-Tree thì đó là <strong>clustered index</strong>. Việc lưu luôn bản ghi cùng với key khiến các bản ghi có key gần nhau thường được lưu gần nhau (gần ở đây là trong cùng 1 page luôn), đó là lí do có từ clustered. Mục đích là để tăng tốc độ tìm và đọc bản ghi.

Mỗi một bảng sẽ chỉ có một clustered index vì nếu không mỗi lần update một bản ghi, storage engine sẽ phải tìm và update ở nhiều nơi khác nhau. Vậy nên các index khác sẽ không lưu bản ghi ở nút lá mà thay vì đó sẽ lưu tham chiếu tới bản ghi hoặc là lưu giá trị của column được đánh clustered index. Một lần nữa, quyết định chọn giải pháp nào là tùy thuộc vào từng storage engine, nhưng ta đều gọi chung các index đó là <strong>secondary index</strong>.
![](https://res.cloudinary.com/vido/image/upload/v1567941148/images/clustered-index/Screenshot_from_2019-09-08_18-11-50_qniblb.png)

Khi sử dụng clustered index thì không nên xét giá trị của cột tương ứng là các giá trị random (như UUID chẳng hạn) vì nó sẽ ảnh hưởng rất nhiều đến tốc độ ghi, thay vào đó nên set attribute AUTO_INCREMENT cho cột đó. Lí do là vì khi bạn insert bản ghi tuần tự từ trái sang phải (do key có giá trị tăng dần) các bản ghi và key được ghi vào từng page, đầy page này thì sang page khác như hình dưới
![](https://res.cloudinary.com/vido/image/upload/v1567953475/images/clustered-index/Screenshot_from_2019-09-08_21-37-18_mj5cxc.png)
Còn nếu giá trị key đó là ngẫu nhiên, thì khi thêm một bản ghi có thể bạn sẽ phải ghi nó vào một page vốn đã đầy. Lúc này storage engine phải tách page cũ, kiếm một page mới và copy nửa bên phải sang page mới rồi mới ghi bản ghi mới vào page cũ. Những thao tác đó tốn rất nhiều thao tác I/O. Không những tốn thời gian, nó còn dẫn đến tình trạng phân mảnh nội (internal fragmentation), nhớ về ví dụ trang giấy ở trên, giống như là không viết hết tất cả các dòng của trang sách, mỗi trang chỉ ghi được một số dòng rồi để thừa dẫn đến ... lãng phí. Kích thước của cây index sẽ phình ra (giống như là quyển sách bị độn dày thêm). 

Dưới đây là benchmark khi chèn thêm bản ghi vào bảng (dùng InnoDB của MySQL) khi xài key tăng dần và UUID. Có thể thấy rõ là bảng càng nhiều bản ghi thì thời gian ghi mới càng lâu và kích thước cây càng lớn.
![](https://res.cloudinary.com/vido/image/upload/v1567954291/images/clustered-index/Screenshot_from_2019-09-08_21-51-07_bhdnmy.png)

### Clustered Index trong MySQL
Với MySQL, bạn cần dùng storage engine InnoDB để có thể dùng clustered index. InnoDB sẽ đánh clustered index trên primary key và trong trường hợp bạn không define primary key thì nó sẽ thử sử dụng một unique nonnullable key. Nếu vẫn không tìm được thì nó sẽ tiếp tục khai báo một primary key ẩn (chính là ID) và đánh clustered index trên key này. 
Secondary Index của InnoDB sẽ lưu key cùng với primary key ở nút lá. Vì vậy nếu query của bạn hit secondary index thì bạn sẽ tốn 2 lần đi trên cây B-Tree. Một lần cho secondary index để tìm được primary key và lần 2 là lấy dữ liệu từ clustered index. Nhưng nếu bạn select toàn các field trong 1 composite index thì sẽ chỉ tốn 1 lần vì các field đó đều đã được lưu trong nút lá rồi, đây gọi là covering index.
Nếu bạn dùng storage engine là InnoDB thì mặc định là sẽ có clustered index trên primary key rồi. Vì vậy nếu việc ghi thêm 1 hàng diễn ra khá nhiều thì hãy set primary key là số và để AUTO_INCREMENT hoặc tự đảm bảo là các key ghi vào sẽ có thứ tự từ điển tăng dần. Cá nhân mình thích cách đầu tiên hơn vì nó tích kiệm được rất nhiều công sức, ta chỉ cần ghi data, storage engine sẽ lo hết phần còn lại.
## Kết Luận
Tổng kết lại, thì lợi ích của việc sử dụng clustered index (hoặc vô tình xài, nếu bạn đang dùng InnoDB của MySQL):

- Truy cập data nhanh hơn vì bản ghi cần tìm được lưu luôn trong nút lá.
- Truy vấn sử dụng covering index có thể trả về luôn các giá trị lưu ở nút lá.
- Các dữ liệu liên quan đến nhau được lưu thành các cụm gần nhau. Ví du, khi phát triển một ứng dụng như google calender chẳng hạn, bằng việc set primary key tinh tế, bạn có thể truy vấn các sự kiện diễn ra trong 1 khu vực chỉ bằng việc đọc vài page.

Tuy nhiên chúng ta cần nắm được các điểm bất lợi của clustered index:

- Tốc độ ghi phụ thuộc vào thứ tự ghi.
- Update clustered index column sẽ tốn rất nhiều disk I/O vì phải di chuyển hoàn toàn các bản ghi được update sang page khác.
- Việc update hoặc chèn thêm bản ghi mới vào sẽ gây ra page split rất nhiều (phải tách page đã đầy bản ghi thành 2 page khác nhau để chèn thêm bản ghi mới). Page split sẽ khiến không gian lưu trữ bị phình ra.
- Truy vấn sử dụng secondary index có thể chậm vì phải lookup index 2 lần.

## Tài liệu tham khảo
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [High Performance MySQL](https://www.highperfmysql.com/)

