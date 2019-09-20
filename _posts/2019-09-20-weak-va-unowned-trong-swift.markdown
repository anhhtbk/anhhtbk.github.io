---
layout: post
title:  "Weak và Unowned trong Swift"
date:   2019-09-20 10:29:26 +0700
categories: iOS
---
Đây là bài dịch lại từ [https://medium.com/better-programming/weak-and-unowned-keywords-in-swift-7bda8bdd97c4](https://medium.com/better-programming/weak-and-unowned-keywords-in-swift-7bda8bdd97c4)
# Weak và Unowned trong Swift
Tầm quan trọng khi sử dụng chúng trong closures & ví dụ

![](https://cdn-images-1.medium.com/max/7232/1*dcIjZwfAqnliUq6FR8tMyg.png)

Nếu bạn quen thuộc với Swift thì chắc hẳn bạn đã nghe đến hai từ khóa *unowned*, *weak*. Nếu bạn chưa từng nghe, thì hãy tìm hiểu chúng bây giờ. Tôi sẽ không giải thích tất cả mọi thứ về những từ khóa này, nhưng tôi sẽ nói cho bạn biết vì sao bạn nên biết và nên sử dụng chúng.

Điều gì sẽ xảy ra nếu chúng ta không dùng unowned, weak trong closures?

Câu trả lời ngắn gọn là: memory leaks!

Câu trả lời dài hơn là: hãy để tôi cho bạn thấy…

## Cách chúng hoạt động!

Để xem việc không sử dụng unowned, weak có thể tạo ra memory leaks như thế nào, tôi đã tạo ra một ứng dụng đơn giản cho phép bạn nhập vào web request status code, sau đó hiển thị chúng trong một tableview với những bức ảnh các chú mèo tương ứng.

Tôi đã không nghĩ ra ý tưởng tuyệt vời nào để ghép những mã lỗi với hình ảnh những chú mèo, nhưng tôi đã khám phá ra API tuyệt vời, và dễ sử dụng:
[**HTTP Cats**](https://http.cat/)


Nó có vẻ như là một API thú vị để sử dụng trong một ứng dụng ví dụ.

## App Setup

![](https://cdn-images-1.medium.com/max/3760/1*gIcu_U55uhKDyFePVoDE9w.png)

Việc thiết lập khá dễ dàng. Khi mở ứng dụng bạn sẽ thấy một màn hình xuất hiện với button "Start". Khi nhấn vào button màn hình chính xuất hiện. Trong màn này bạn có thể nhập vào một status code gồm 3 số và nhấn "Find". Tất cả những status code đã được nhập sẽ hiển thị ở tableview bên dưới. Nếu một bức ảnh được tìm thấy từ [http.cat/[status code]](https://http.cat/404) thì nó sẽ được hiển thị với status code tương ứng.
## Data Model

Data model khá là đơn giản. Chúng ta có 1 class CatStatus. Class này bao gồm các variables: value (giá trị status code), imageData; 1 function tên là updated, được gọi khi API kết thúc và ảnh đã tải xong; và function makeRequest() tạo request tới API http.cat để tìm ảnh 1 chú mèo tương ứng với 1 status code.
```swift
import Foundation

class CatStatus {
    let value: String
    var imageData: Data?
    var updated: (()->())?

    init(value: String) {
        self.value = value
    }

    func makeRequest() {
        DispatchQueue.global().async {
            if let url = URL(string: "https://http.cat/\(self.value)") {
                if let data = try? Data(contentsOf: url) {
                    self.imageData = data

                    if let updated = self.updated {
                        updated()
                    }
                }
            }
        }
    }
}
```
## View Controller

Khi người dùng nhấn vào button “Find” thì function sẽ được thực thi trong view controller:
```swift
@IBAction func didSubmit() {
    if let text = self.textField?.text {
        if text.count == 3 {
            self.textField?.textColor = .black // status code is valid
            self.textField?.text = nil

            let newCat = CatStatus(value: text)
            self.cats.append(newCat)

            let indexPath = IndexPath(row: cats.count - 1, section: 0)
            tableView?.insertRows(at: [indexPath], with: .automatic)

            newCat.updated = { [unowned self] in
                DispatchQueue.main.async {
                    self.tableView?.reloadRows(at: [indexPath], with: .automatic)
                }
            }
            newCat.makeRequest()
        } else {
            self.textField?.textColor = .red // status code has more or less than 3 digits
        }
        if text.count == 0 {
            self.textField?.textColor = .black // status code is empty
        }
    }
}
```
Trước tiên, chúng ta kiểm tra xem văn bản có nil không và độ dài văn bản là 3 (điều này có nghĩa là chúng ta có thể có status code hợp lệ) và sẽ thêm một row mới vào tableview. Chúng ta tạo một instance mới của CatStatus và thêm nó vào danh sách mèo của chúng ta. Sau đó, chúng ta nói với tableview rằng một row mới cần được thêm vào.

Bây giờ đến phần thú vị: chúng ta muốn nói với CatStatus mới thực hiện một request API. Trước tiên chúng ta tạo một closure gán vào variable updated để chúng ta được thông báo khi imageData được cập nhật. Nếu imageData đã được cập nhật, chúng ta muốn tableview tải lại row tương ứng.

Sau cùng là makeRequest để bắt đầu gọi đến API
## Unowned và Weak

Đây là phần thú vị. Như bạn thấy [unowned self] hiện tại đang bị comment out. Nhưng khác biệt là gì khi chúng ta viết [unowned self] hoặc không viết? Hãy run app và bạn sẽ thấy. Sau đó tôi sẽ giải thích.
```swift
newCat.updated = { // [unowned self] in
    DispatchQueue.main.async {
        self.tableView?.reloadRows(at: [indexPath], with: .automatic)
    }
}
```
## Thử nghiệm

![](https://cdn-images-1.medium.com/max/2000/1*VS2HaLnajotIL9seskjj8A.png)

Tôi chạy ứng dụng 2 lần. Lần đầu tiên [unowned self] bị comment out còn lần sau thì không. Ngoài ra thì không có điều gì thay đổi. Bạn sẽ thấy sự khác nhau trong memory usage.

Các bước thực hiện:

1. Mở ứng dụng

1. Nhấn “Start”

1. Tìm status code = “100”

1. Tìm status code = “200”

1. Tìm status code = “300”

1. Tìm status code = “400”

1. Nhấn “Back”

1. Lặp lại các bước từ 2 đến 7 thêm ba lần nữa

### Lần chạy đầu tiên

Lần chạy đầu tiên với [unowned self] bị comment out cho kết quả memory usage như sau:

![](https://cdn-images-1.medium.com/max/2044/1*oFRPgDwNhVjXXDGYWzsebA.png)

Như bạn thấy, memory usage tăng đều đặn. Mỗi lần một ảnh được truy vấn memory lại tăng thêm. Điều quan trọng cần lưu ý là không phải tất cả các hình ảnh đều được hiển thị cùng một lúc. Khi chúng ta trở lại màn hình đầu tiên dữ liệu từ các truy vấn trước đã mất, nhưng không được release.

### Lần chạy thứ 2

Lần chạy thứ 2 với [unowned self] không bị comment out:

![](https://cdn-images-1.medium.com/max/2020/1*pFKFBBCKO_NM7OHnfI4g2A.png)

Như bạn thấy, mỗi lần chúng ta trở lại màn hình chính thì memory của những ảnh đã truy vấn sẽ được giải phóng.

## Giải thích

Tại sao lần chạy đầu tiên memory không được giải phóng?
```
newCat.updated = { // [unowned self] in
    DispatchQueue.main.async {
        self.tableView?.reloadRows(at: [indexPath], with: .automatic)
    }
}
```
Swifts tự động quản lý memory:

* mỗi lần chúng ta tạo 1 strong reference, reference count sẽ tăng lên 1

* mỗi lần chúng ta giải phóng 1 strong reference, reference count sẽ giảm đi 1

Đây là cách để swift biết nếu 1 object vẫn còn tham chiếu bởi object khác và sẽ không bị giải phóng. Nếu reference count = 0, thì object sẽ được giải phóng.

Điều gì xả ra trong 1 closure khi chúng ta tham chiếu đến self, chugns ta tạo một strong reference, vì vậy reference count tăng lên 1. Bây giờ khi chúng ta quay lại màn hình trước thì điều gì sẽ ra:
* ViewController sẽ được giải phóng, nếu như chúng ta không tham chiếu đến nó từ bất cứ đâu

* Tất cả CatStatus sẽ được giải phóng, nếu như chúng ta không tham chiếu đến chúng từ bất cứ đâu

* Tất cả các biến imageData sẽ được giải phóng, nếu như chúng ta không tham chiếu đến chúng từ bất cứ đâu

Điều đó đã không xảy ra, vì closure vẫn giữ 1 tham chiếu đến ViewController (ở đây là: self). Điều đó có nghĩa là ViewController không được giải phóng. Vì vậy vẫn còn những tham chiếu đến CatStatus và vẫn còn những tham chiếu đến biến imageData.

Kết quả là rất nhiều memory đang được sử dụng mà chúng ta không thể truy cập tới.

Giải pháp: sử dụng [unowned self] hoặc [weak self] trong closure. Cách này chúng ta vẫn có thể tham chiếu đến self, nhưng không tạo ra một strong reference.

## Unowned hay Weak?

Khi chúng ta không sử dụng strong reference thì self có thể sẽ được giải phóng khi kết thúc closure.

Khi chúng ta sử dụng unowned nếu self đã được giải phóng, thì ứng dụng sẽ crash. Vì thế chỉ sử dụng unowned nếu bạn chắc chắn 100% rằng điều đó sẽ không xảy ra.

Khi chúng ta sử dụng weak nếu self đã được giải phóng, thì self = nil. Điều này có nghĩa là khi sử dụng weak, self sẽ là một optional. Điều này cho phép chúng ta kiểm xoát trường hợp self đã bị giải phóng.

Trong hầu hết các trường hợp, chúng ta sử dụng weak, vì nó an toàn hơn. Tôi đã sử dụng unown trong ví dụ này vì nó có nghĩa là tôi cần thay đổi ít mã hơn.

## Kết luận

Mặc dù Swift giúp việc quản lý bộ nhớ dễ dàng hơn rất nhiều so với  Objective-C, nhưng nó vẫn quan trọng để xem xét nó và để hiểu cách thức tham chiếu hoạt động.

Tôi đã sử dụng một ứng dụng hạn chế và đơn giản để chỉ ra cách rò rỉ bộ nhớ có thể xảy ra, trong trường hợp này, dữ liệu hình ảnh không được giải phóng đã tăng bộ nhớ sử dụng rất nhanh. Trong các ứng dụng trong thế giới thực, nơi một ứng dụng có thể chạy trong một thời gian dài, thậm chí các rò rỉ nhỏ hơn có thể tạo ra các vấn đề trong khung thời gian dài hơn.

Hãy coi điều này như một động lực để tìm hiểu về unowned và weak nếu bạn chưa từng làm điều đó.

Bạn có thể tìm thấy code của ứng dụng ví dụ trên ở đây:
[kristiinara/MemoryleaksTestApp](https://github.com/kristiinara/MemoryleaksTestApp)

Có những bài viết tuyệt vời giải thích chi tiết về unowned and weak, ví dụ như:
[Swift Retention Cycle in Closures and Delegate. Let’s understand [weak self], [unowned self] , and weak var](https://blog.bobthedeveloper.io/swift-retention-cycle-in-closures-and-delegate-836c469ef128)
