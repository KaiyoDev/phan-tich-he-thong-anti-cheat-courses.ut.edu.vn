# Phân tích Cơ chế Secure Quiz

README này được viết theo hướng một bản ghi chép nghiên cứu kỹ thuật ngắn, nhằm phân tích cơ chế hoạt động của trang quiz bảo mật được mô phỏng trong `demo-tracnghiem.html`, đối chiếu với mẫu trang gốc đã phân tích, và trình bày cách kiểm thử hợp lệ trên môi trường local. Tài liệu phục vụ mục đích nghiên cứu học tập và phòng vệ; nội dung được giới hạn ở phân tích client-side, mô phỏng local, và các đề xuất cải tiến telemetry, không hướng dẫn vượt cơ chế kiểm soát trên hệ thống thật như [courses.ut.edu.vn](https://courses.ut.edu.vn/).

## Tóm tắt nhanh

- Trang quiz gồm hai lớp chính: lớp giao diện/theme của Moodle ở phần `head` và lớp kiểm soát hành vi ở phần JavaScript.
- Các hành vi như rời tab, chuột phải, copy/cut/paste, bôi đen và phím tắt clipboard đều có thể bị chặn ở phía client.
- `demo-tracnghiem.html` giữ lại bố cục Moodle/YUI và mô phỏng các cơ chế chặn để phục vụ kiểm thử local.
- Bảng `Hành vi người dùng` trong sidebar là một lớp telemetry UI đơn giản; nếu triển khai trên hệ thống thật, nên bổ sung log có cấu trúc và biểu diễn nhị phân để tổng hợp sự kiện hiệu quả hơn.

## 1. Bối cảnh hệ thống

Trang đích gốc thuộc hệ thống đào tạo trực tuyến Moodle của trường, với branding `Hệ thống đào tạo trực tuyến` thể hiện trên trang đăng nhập [courses.ut.edu.vn](https://courses.ut.edu.vn/). Bản demo local được dùng để:

- giữ giao diện gần với trang quiz thật;
- quan sát cách event bị chặn trên trình duyệt;
- thử nghiệm telemetry và UI mà không cần tác động lên hệ thống đang vận hành.

## 2. Kiến trúc tổng quát

Có thể xem trang quiz bảo mật gồm ba lớp:

```mermaid
flowchart TD
  userAction[UserAction]
  clientUi[ClientUI]
  eventLayer[EventLayer]
  alertBlock[AlertOrBlock]
  telemetry[Telemetry]

  userAction --> clientUi
  clientUi --> eventLayer
  eventLayer --> alertBlock
  eventLayer --> telemetry
```

### 2.1 Lớp nạp tài nguyên và khởi tạo môi trường

Phần `head` của trang gốc nạp CSS và bootstrap YUI/Moodle, bao gồm:

- stylesheet YUI/Moodle;
- `firstthemesheet` để làm mốc cho hệ thống chèn CSS động;
- `M.cfg`, `YUI_config`, `M.yui.loader`;
- các script theme/head để body được gán thêm class `jsenabled`.

Trong `demo-tracnghiem.html`, cấu trúc này đã được đưa về gần bản gốc nhất có thể. Vì vậy, các phần như:

- `navbar`;
- `#page`, `#page-header`, `#region-main`;
- `#mod_quiz_navblock`;
- các class như `card`, `qnbutton`, `quiz-timer-inner`

đều phụ thuộc trực tiếp vào CSS của theme Moodle.

Một chi tiết quan trọng cần nhấn mạnh là pipeline nạp JavaScript ở đầu trang. Trong `demo-tracnghiem.html`, cụm script sau thể hiện rõ trình tự khởi tạo front-end:

```41:44:demo-tracnghiem.html
</div><script src="https://courses.ut.edu.vn/lib/javascript.php/1774818002/lib/polyfills/polyfill.js"></script>
<script src="https://courses.ut.edu.vn/theme/yui_combo.php?rollup/3.18.1/yui-moodlesimple-min.js"></script><script src="https://courses.ut.edu.vn/theme/jquery.php/core/jquery-3.7.1.min.js"></script>
<script src="https://courses.ut.edu.vn/lib/javascript.php/1774818002/lib/javascript-static.js"></script>
<script src="https://courses.ut.edu.vn/theme/javascript.php/edly/1774818002/head"></script>
```

Cụm này có thể được hiểu như một pipeline bốn tầng:

- `polyfill.js`: tạo lớp tương thích cho trình duyệt, bảo đảm các API cần thiết tồn tại trước khi các module cao hơn chạy.
- `yui-moodlesimple-min.js`: nạp runtime YUI/Moodle, là lớp hạ tầng cho các module cũ của Moodle và một số utility về DOM/event.
- `jquery-3.7.1.min.js`: bổ sung lớp tiện ích quen thuộc cho plugin và theme; dù không phải phần nào cũng cần jQuery, nó vẫn là dependency quan trọng của hệ thống.
- `javascript-static.js` và `theme/.../head`: nạp logic chung của Moodle và logic theme `edly`, giúp class body, theme behavior, utility và các thành phần UI được đồng bộ.

Nếu nhìn dưới góc độ kiến trúc, đây không chỉ là “nạp thêm script”, mà là một chuỗi khởi tạo có thứ tự. Thứ tự này quan trọng vì:

- theme có thể phụ thuộc vào utility đã được nạp trước;
- module chặn hành vi chỉ ổn định khi hạ tầng DOM/event đã sẵn sàng;
- thay đổi bất kỳ mắt xích nào cũng có thể dẫn đến lệch giao diện hoặc sai hành vi.

### 2.2 Lớp giao diện và nội dung bài thi

Trong `demo-tracnghiem.html`, phần nội dung hiện có:

- một câu trắc nghiệm demo;
- một câu tự luận demo;
- bảng điều hướng câu hỏi ở sidebar;
- bảng theo dõi `Hành vi người dùng`.

Đoạn mã thể hiện câu 1 và câu 2 hiện đang ở file demo:

```72:90:demo-tracnghiem.html
<div id="page-content" class="row">
    <div id="region-main-box" class="col-12">
        <section id="region-main" class="has-blocks" aria-label="Nội dung">
            ...
            <div id="question-3989209-1" class="que multichoice deferredfeedback notyetanswered">
                ...
                <div class="qtext"><p>Đây là câu hỏi demo trắc nghiệm số 1.</p></div>
                ...
```

```90:90:demo-tracnghiem.html
</div><div id="question-3989209-2" class="que essay deferredfeedback notyetanswered"><div class="info"><h3 class="no">Câu hỏi <span class="qno">2</span></h3>...
```

Ý nghĩa của các class/chunk chính:

- `que multichoice`: câu trắc nghiệm một lựa chọn;
- `que essay`: câu tự luận;
- `questionflag editable`: vùng đánh dấu câu hỏi;
- `submitbtns`: vùng điều hướng sang bước tiếp theo.

Ở mức cốt lõi, lớp giao diện này đóng vai trò:

- hiển thị trạng thái câu hỏi;
- làm bề mặt tương tác cho người dùng;
- tạo ra ngữ cảnh DOM để lớp JavaScript có thể bắt sự kiện và áp chính sách.

### 2.3 Lớp bắt sự kiện và ghi nhận hành vi

Phần JavaScript client-side trong bản demo đang chặn và ghi nhận các sự kiện:

```154:219:demo-tracnghiem.html
<script src="https://courses.ut.edu.vn/lib/javascript.php/1774818002/lib/requirejs/require.min.js"></script>
<script>
//<![CDATA[
document.getElementById("responseform").addEventListener("submit", function(e) { e.preventDefault(); });
document.getElementById("quiz-time-left").textContent = "13:23";
var behaviorCounts = {
    visibility: 0,
    contextmenu: 0,
    copy: 0,
    paste: 0,
    selectstart: 0
};
...
document.addEventListener("selectstart", function(e) {
    e.preventDefault();
    behaviorCounts.selectstart += 1;
    updateBehaviorCell("selectstart");
});
//]]>
</script>
```

Nếu xem kỹ đoạn mở đầu của khối này:

```158:166:demo-tracnghiem.html
document.getElementById("responseform").addEventListener("submit", function(e) { e.preventDefault(); });
document.getElementById("quiz-time-left").textContent = "13:23";
var behaviorCounts = {
    visibility: 0,
    contextmenu: 0,
    copy: 0,
    paste: 0,
    selectstart: 0
};
```

Ta thấy ba ý tưởng thiết kế rất rõ:

- `preventDefault()` trên form: biến trang thành một môi trường test quan sát, tránh submit thật và giúp các thao tác trong UI được lặp lại nhiều lần.
- gán sẵn `quiz-time-left`: tách riêng mục đích mô phỏng UI timer khỏi logic timer thật; điều này giữ giao diện “sống” mà không cần backend.
- `behaviorCounts`: sự kiện không chỉ bị chặn, mà còn được biến thành dữ liệu có thể đo đếm. Đây là bước chuyển từ “chặn hành vi” sang “quan sát hành vi”.

Dưới góc nhìn nghiên cứu, `behaviorCounts` là một telemetry accumulator rất nhỏ. Nó không phức tạp, nhưng cho thấy một nguyên lý quan trọng: hệ thống client-side hiệu quả hơn khi không chỉ cản trở, mà còn lưu vết.

Những nhóm hành vi hiện đang được xử lý là:

- `visibilitychange`: cảnh báo khi rời tab;
- `contextmenu`: chặn chuột phải;
- `copy`, `cut`, `paste`: chặn thao tác clipboard;
- `dragstart`: chặn kéo thả;
- `selectstart`: chặn bôi đen;
- `keydown` kết hợp `Ctrl`/`Meta`: chặn phím tắt clipboard.

## 3. Cơ chế hoạt động cốt lõi

Đây là phần quan trọng nhất của tài liệu. Nếu rút gọn toàn bộ trang thành chuỗi vận hành cốt lõi, ta có thể mô tả theo sáu giai đoạn:

1. Trang tải tài nguyên.
2. DOM được dựng xong.
3. Listener được đăng ký.
4. Người dùng phát sinh hành vi.
5. Hệ thống chặn hoặc cảnh báo.
6. Telemetry/UI được cập nhật.

Sơ đồ dưới đây minh họa chu trình phản ứng:

```mermaid
flowchart TD
  browserEvent[BrowserEvent]
  listenerCheck[ListenerCheck]
  preventStep[PreventDefault]
  counterUpdate[CounterUpdate]
  uiUpdate[UIUpdate]
  alertStep[AlertStep]

  browserEvent --> listenerCheck
  listenerCheck --> preventStep
  preventStep --> counterUpdate
  counterUpdate --> uiUpdate
  counterUpdate --> alertStep
```

### 3.1 Giai đoạn 1: Tải tài nguyên

Trình duyệt lần lượt tải:

- CSS nền và theme;
- runtime YUI/Moodle;
- jQuery;
- JavaScript tĩnh và script theme.

Đây là giai đoạn tạo “nền” cho mọi phần phía sau. Nếu giai đoạn này sai:

- giao diện sẽ lệch;
- class/body có thể không khởi tạo đúng;
- các listener phía sau có thể hoạt động thiếu ổn định.

### 3.2 Giai đoạn 2: Dựng DOM

Sau khi tài nguyên được nạp, HTML tạo ra:

- vùng nội dung chính;
- form bài thi;
- câu hỏi trắc nghiệm;
- câu tự luận;
- bảng điều hướng câu hỏi;
- bảng hành vi người dùng.

Đây là điều kiện cần để script cuối trang có thể tìm thấy:

- `responseform`
- `quiz-time-left`
- các ô đếm hành vi như `user-behavior-copy`

### 3.3 Giai đoạn 3: Đăng ký listener

Khi script cuối trang chạy, hệ thống:

- chặn submit form;
- gán giá trị ban đầu cho timer;
- tạo đối tượng `behaviorCounts`;
- gắn hàng loạt listener vào `document`.

Điểm quan trọng là phần này diễn ra sau khi DOM đã hiện diện, nên thao tác `getElementById(...)` hoạt động trực tiếp, không cần thêm bootstrap phức tạp.

### 3.4 Giai đoạn 4: Phát sinh hành vi người dùng

Người dùng có thể:

- rời tab;
- bấm chuột phải;
- copy/paste;
- bôi đen;
- dùng tổ hợp phím clipboard.

Mỗi hành vi như vậy đi vào cùng một mô hình xử lý:

- listener bắt được sự kiện;
- script kiểm tra điều kiện;
- `preventDefault()` nếu cần;
- tăng bộ đếm;
- cập nhật giao diện;
- hiển thị cảnh báo nếu chính sách yêu cầu.

### 3.5 Giai đoạn 5: Chặn hoặc cảnh báo

Không phải hành vi nào cũng bị xử lý giống nhau:

- `visibilitychange`: thiên về cảnh báo khi điều kiện `document.hidden` đúng;
- `contextmenu`, `copy`, `paste`: vừa chặn vừa cảnh báo;
- `selectstart`: chặn và ghi nhận, không nhất thiết phải bật `alert`.

Điều này cho thấy trang đang dùng nhiều mức phản hồi:

- mức 1: chặn im lặng;
- mức 2: chặn và thông báo;
- mức 3: chặn, thông báo, và ghi nhận telemetry.

### 3.6 Giai đoạn 6: Cập nhật telemetry

Sau khi chặn sự kiện, hệ thống cập nhật bộ đếm và phản ánh ngay trong bảng `Hành vi người dùng`.

Đây là bước có giá trị nghiên cứu lớn nhất, vì nó biến hành vi tức thời thành dữ liệu có thể:

- nhìn thấy;
- thống kê;
- so sánh;
- tổng hợp thành mẫu.

## 4. Phân tích chi tiết từng lớp kiểm soát

### 4.1 Cảnh báo rời tab

Khi `document.hidden === true`, script:

- tăng bộ đếm `visibility`;
- cập nhật bảng hành vi người dùng;
- hiện `alert`.

Tác dụng:

- có thể phát hiện người dùng chuyển tab, ẩn cửa sổ, hoặc mất focus theo cách trình duyệt đánh dấu;
- dễ triển khai;
- nhưng dễ gây phiền toái do `alert` khóa luồng tương tác.

Hạn chế:

- hoàn toàn phía client;
- có thể phát sinh false positive;
- không đủ để kết luận hành vi bất thường nếu không kết hợp telemetry khác.

### 4.2 Chặn chuột phải

Sự kiện `contextmenu` bị `preventDefault()` và hiển thị cảnh báo.

Ý nghĩa:

- ngăn menu chuột phải mặc định;
- làm giảm một số thao tác copy nhanh;
- nhưng không nên xem là cơ chế bảo mật mạnh, vì nó chỉ tác động lên lớp UI của trình duyệt.

### 4.3 Chặn copy/cut/paste

Với `copy`, `cut`, `paste`, trang:

- chặn event;
- tăng bộ đếm tương ứng nếu có cấu hình ghi nhận;
- hiện thông báo cho người dùng.

Tác dụng thực tế:

- giúp người dùng hiểu rằng hệ thống đang áp dụng chính sách hạn chế;
- đồng thời cung cấp dữ liệu về tần suất cố gắng thao tác clipboard.

### 4.4 Chặn bôi đen văn bản

Với `selectstart`, trang:

- chặn event;
- tăng bộ đếm `selectstart`;
- thường không cần thông báo bằng `alert`.

Trong các hệ thống quiz, đây là cách phổ biến để giảm thao tác copy từ chọn văn bản. Tuy vậy, giá trị đáng chú ý hơn là việc hành vi này được ghi nhận như một tín hiệu.

### 4.5 Chặn phím tắt clipboard

Script bắt `keydown` và kiểm tra:

- `Ctrl + C`;
- `Ctrl + V`;
- `Ctrl + X`;
- hoặc `Meta` trên hệ điều hành có phím command.

Nếu khớp:

- `preventDefault()`;
- hiển thị cảnh báo tính năng bị vô hiệu hóa.

Lớp này cho thấy logic kiểm soát không chỉ nhìn vào thao tác chuột mà còn bao phủ cả luồng bàn phím.

## 5. Bảng câu hỏi và sidebar

Khung `Bảng câu hỏi` là một phần quan trọng của bố cục và là nơi dễ phát sinh sai lệch CSS nếu không giữ đúng DOM/class Moodle.

Trong demo hiện tại, cấu trúc card phải đã được giữ sát bản gốc:

```100:118:demo-tracnghiem.html
<section data-region="blocks-column" aria-label="Các khối">
    <aside id="block-region-side-pre" class="block-region" data-blockregion="side-pre" data-droptarget="1"><h2 class="sr-only">Các khối</h2><a href="#sb-1" class="sr-only sr-only-focusable">Bỏ qua Bảng câu hỏi</a>

<section id="mod_quiz_navblock"
     class=" block block_fake  card mb-3"
     role="navigation"
...
```

Và bên trong card:

- `#quiznojswarning`;
- `qn_buttons clearfix multipages`;
- `othernav`;
- bảng `Hành vi người dùng`.

Bảng `Hành vi người dùng` là phần có giá trị nghiên cứu cao nhất của bản demo hiện tại, vì nó đóng vai trò như một lớp telemetry trực quan trong UI:

- người xem có thể kiểm tra event nào xảy ra nhiều;
- có thể đối chiếu giữa hành vi thật và thông báo của trang;
- có thể sử dụng bảng này như một dashboard mini trong local test.

Nếu bố cục/card phải bị lệch màu nền, nguyên nhân phổ biến thường là:

- bổ sung CSS custom không cần thiết;
- thay đổi DOM quanh `card-body`, `card-text`;
- chèn thêm phần tử mới không theo spacing của theme.

## 6. So sánh trang gốc và bản demo

### 6.1 Điểm giống

- cùng bố cục header + main region + blocks column;
- cùng tên class Moodle quan trọng;
- cùng pattern chặn event phía client;
- cùng sidebar điều hướng câu hỏi;
- cùng timer và submit form.

### 6.2 Điểm mô phỏng

- nội dung câu hỏi đã được đổi thành `demo`;
- một số hidden input vẫn giữ kiểu dữ liệu gốc để phục vụ giao diện;
- script cuối file đã được giản lược cho local, nhưng vẫn giữ logic chặn event và đo đếm hành vi;
- bảng `Hành vi người dùng` là phần phục vụ quan sát, không phải thành phần gốc của trang thật.

## 7. Vì sao bảng nhị phân trong console hiệu quả hơn

Bảng đếm UI hữu ích để xem nhanh, nhưng nếu muốn tổng hợp hành vi hiệu quả hơn thì nên bổ sung biểu diễn nhị phân (bitmask) trong `console` hoặc telemetry. Dưới góc nhìn nghiên cứu, đây là bước chuyển từ “quan sát thủ công” sang “biểu diễn máy đọc được”.

### 7.1 Ý tưởng

Gán mỗi hành vi một bit:

- `1 << 0`: rời tab
- `1 << 1`: chuột phải
- `1 << 2`: copy
- `1 << 3`: paste
- `1 << 4`: bôi đen
- `1 << 5`: cut
- `1 << 6`: dragstart
- `1 << 7`: clipboard shortcut

Khi có hành vi xảy ra:

- bật bit tương ứng;
- ghi log thêm timestamp và lần xuất hiện.

Ví dụ:

```js
const BehaviorBits = {
  visibility: 1 << 0,
  contextmenu: 1 << 1,
  copy: 1 << 2,
  paste: 1 << 3,
  selectstart: 1 << 4
};

let behaviorMask = 0;
behaviorMask |= BehaviorBits.copy;
console.log("behaviorMask", behaviorMask.toString(2).padStart(8, "0"));
```

### 7.2 Lợi ích

- giảm chi phí tổng hợp trạng thái hiện tại;
- dễ lưu telemetry gọn hơn;
- dễ so khớp pattern bất thường;
- dễ vẽ dashboard/báo cáo theo phiên;
- giúp server-side rule engine xử lý nhanh hơn nếu cần.

Nếu viết theo phong cách nghiên cứu hệ thống, biểu diễn nhị phân có ba ưu điểm nổi bật:

- `Tính cố định`: mỗi bit có nghĩa rõ ràng, tránh mơ hồ hoặc log text tự do.
- `Tính tổng hợp`: có thể gộp nhiều hành vi vào một giá trị trạng thái duy nhất.
- `Tính so sánh`: dễ đối chiếu giữa các phiên, người dùng, và mốc thời gian.

### 7.3 Khi nào dùng bảng đếm trực quan, khi nào dùng bitmask

- Bảng UI: dùng để nhìn nhanh trong local/demo, phục vụ manual testing.
- Bitmask/console log: dùng để tổng hợp, lưu vết, và phân tích hành vi theo phiên.
- Hệ thống thật nên dùng cả hai: UI nhẹ cho debug nội bộ, bitmask + timestamp cho telemetry.

## 8. Checklist kiểm thử hợp lệ trên local/demo

Tất cả bài test dưới đây chỉ nên thực hiện trên `demo-tracnghiem.html`.

### 8.1 Kiểm thử giao diện

- mở `demo-tracnghiem.html`;
- xác nhận header, timer, `Bảng câu hỏi`, và hai câu hỏi hiển thị đúng;
- xác nhận bảng `Hành vi người dùng` nằm dưới `Làm xong ...`.

### 8.2 Kiểm thử rời tab

- chuyển sang tab khác;
- quay lại;
- xác nhận có cảnh báo và bộ đếm `Rời tab` tăng.

### 8.3 Kiểm thử chuột phải

- bấm chuột phải trong trang;
- xác nhận menu mặc định không mở;
- bộ đếm `Chuột phải` tăng.

### 8.4 Kiểm thử copy/paste/bôi đen

- thử bôi đen văn bản;
- thử `Ctrl + C`, `Ctrl + V`;
- xác nhận bộ đếm trong bảng tăng đúng theo từng loại.

### 8.5 Kiểm thử trắc nghiệm và tự luận

- chọn một đáp án ở câu 1;
- nhập văn bản vào câu 2;
- nhấn `Trang tiếp`;
- xác nhận form không submit thật trong local demo.

### 8.6 Kiểm thử userscript local

Nếu có sử dụng userscript local, chỉ nên dùng trên file demo local. Không mở rộng sang domain thật. Cần kiểm tra:

- script có đúng file `@match`;
- có thể bật/tắt công tắc bypass local;
- khi tắt bypass, page trở lại hành vi chặn mặc định;
- khi bật bypass, page cho phép test UX không bị `alert` làm gián đoạn.

## 9. Đề xuất cải tiến phòng vệ hiệu quả hơn

Nếu xây dựng hệ thống thật, không nên dựa hoàn toàn vào client-side blocking.

### 9.1 Telemetry có timestamp

Mỗi sự kiện nên ghi:

- `attemptId`
- `userId`
- `eventType`
- `timestamp`
- `page`
- `questionId`
- `maskAfterEvent`

### 9.2 Tổng hợp theo phiên

Cần theo dõi:

- tần suất rời tab;
- chuỗi sự kiện sát nhau trong thời gian ngắn;
- hành vi lặp lại có hệ thống;
- sự khác biệt giữa người dùng thường và phiên bất thường.

### 9.3 Không chặn UI như cách duy nhất

Chặn `copy/paste/contextmenu` chỉ nên là lớp cản trở UI. Các quyết định nghiệp vụ quan trọng nên dựa trên:

- telemetry;
- quy tắc server-side;
- lịch sử phiên;
- đánh giá rủi ro tổng hợp.

### 9.4 Tách chế độ research/demo

Nên có hai mode rõ ràng:

- `production mode`: ghi telemetry, thông báo người dùng, và tắt các công cụ debug;
- `demo/test mode`: cho phép theo dõi, in log, bật UI table, và sử dụng test script chỉ trong môi trường kiểm thử.

## 10. Kết luận

Cơ chế secure quiz được mô phỏng trong dự án này cho thấy một mẫu thiết kế quen thuộc:

- layout Moodle/YUI giữ vai trò hiển thị và bố cục;
- JavaScript client-side xử lý event để cảnh báo và chặn thao tác;
- telemetry có thể được nâng cấp đáng kể nếu dùng thêm bitmask/binary flags và log có timestamp.

Với mục tiêu học tập, `demo-tracnghiem.html` là môi trường phù hợp để:

- đọc và hiểu luồng xử lý event;
- kiểm thử UI/UX của chế độ secure;
- đánh giá điểm mạnh, điểm yếu của client-side blocking;
- thử nghiệm hướng telemetry phòng vệ hiệu quả hơn.

## 11. Góc nhìn nghiên cứu

Nếu xem tài liệu này như một nghiên cứu nhỏ về cơ chế secure quiz client-side, có thể rút ra bốn nhận xét:

### 11.1 Đây là một kiến trúc thiên về cản trở hơn là xác minh

Phần lớn cơ chế hiện tại tập trung vào:

- chặn thao tác;
- cảnh báo người dùng;
- làm tăng chi phí thao tác.

Nó chưa phải là một kiến trúc xác minh mạnh, vì các quyết định quan trọng vẫn cần telemetry và logic server-side.

### 11.2 Giá trị lớn nhất nằm ở dấu vết hành vi

Đoạn `behaviorCounts` trong `demo-tracnghiem.html` là một minh họa rất rõ: ngay khi hệ thống có thể biến event thành dữ liệu, nó bắt đầu có giá trị phân tích cao hơn rất nhiều so với việc chỉ hiện `alert`.

### 11.3 UI cản trở và telemetry nên tách vai trò

- UI cản trở: giúp người dùng nhận ra policy.
- Telemetry: giúp hệ thống đánh giá rủi ro.

Nếu trộn hai vai trò vào một, hệ thống dễ vừa gây phiền toái cho người dùng vừa kém hiệu quả trong phân tích.

### 11.4 Bản demo có giá trị học tập cao

Việc giữ bố cục sát bản gốc nhưng đổi nội dung thành `demo` giúp:

- phân tích code dễ hơn;
- thử nghiệm event an toàn hơn;
- viết tài liệu kỹ thuật rõ ràng hơn;
- tách dữ liệu nghiên cứu khỏi nội dung thật của hệ thống.
