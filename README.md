<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ระบบส่องปลาอัจฉริยะ</title>
    <style>
        body, html {
            margin: 0; padding: 0; height: 100%;
            font-family: 'Kanit', sans-serif;
            background: #fdf2f8; /* สีชมพูอ่อนๆ ดูใจดี */
        }

        nav {
            position: fixed; top: 0; width: 100%;
            background: #db2777; color: white;
            display: flex; justify-content: center; padding: 15px 0;
            z-index: 1000;
        }

        nav a { color: white; text-decoration: none; margin: 0 20px; cursor: pointer; }

        section {
            display: none; min-height: 100vh; width: 100%;
            flex-direction: column; justify-content: center; 
            align-items: center; text-align: center; padding: 80px 20px 20px; box-sizing: border-box;
        }

        section.active { display: flex; }

        .btn-upload {
            display: inline-block; padding: 15px 40px;
            cursor: pointer; background: #be185d; color: white;
            border-radius: 12px; font-size: 1.3rem; border: none;
        }

        /* ส่วนแสดงรูปภาพที่อัปโหลด */
        #display-image {
            margin-top: 20px;
            max-width: 300px;
            border-radius: 15px;
            border: 5px solid #fff;
            box-shadow: 0 10px 15px rgba(0,0,0,0.1);
            display: none; /* ซ่อนไว้จนกว่าจะมีการเลือกรูป */
        }

        #result-box {
            margin-top: 20px; padding: 25px;
            border-radius: 20px; background: white;
            width: 90%; max-width: 400px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.05);
        }

        #fish-name { font-size: 2.2rem; font-weight: bold; color: #9d174d; }
        #joke-text { font-size: 1.2rem; color: #475569; margin-top: 15px; line-height: 1.5; }
        
        h1 { color: #831843; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Kanit:wght@300;500&display=swap" rel="stylesheet">
</head>
<body>

    <nav>
        <a onclick="showPage('home')">หน้าแรก</a>
        <a onclick="showPage('check')">ส่องปลา</a>
        <a onclick="showPage('about')">ผู้จัดทำ</a>
    </nav>

    <section id="home" class="active">
        <h1>🐟 ระบบส่องปลา 5G</h1>
        <p>แม่นยำกว่าหมอดู ก็ AI ตัวนี้นี่แหละ!</p>
    </section>

    <section id="check">
        <h1>เอาล่ะ ปลาอะไรเอ่ย?</h1>
        
        <img id="display-image" src="" alt="รูปปลาของคุณ">

        <div id="result-box" style="display: none;">
            <div id="fish-name">...</div>
            <div id="joke-text">...</div>
        </div>

        <div style="margin-top: 20px;">
            <label for="image-upload" class="btn-upload">เลือกภาพ</label>
            <input type="file" id="image-upload" accept="image/*" onchange="processImage(event)" style="display: none;">
        </div>
    </section>

    <section id="about">
        <h1>เกี่ยวกับเรา</h1>
        <p>สร้างโดยคนไทย เพื่อปลาไทย โดยใช้ปัญญาประดิษฐ์</p>
    </section>

    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest/dist/tf.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@latest/dist/teachablemachine-image.min.js"></script>

    <script>
        const URL = "https://teachablemachine.withgoogle.com/models/kjePmuygi/";
        let model;

        const jokes = {
            "ปลานิล": [
                "ควย",
                "ควยยยยยยยยยยยยยยยยยยยยยย",
                "อย่าสบตาน้อง น้องเป็นปลานิลขี้อาย",
                "ปลานิลตัวนี้ดูมีฐานะนะ ดูสิ...เกล็ดเป็นเงินเป็นทอง (ในฝัน)"
            ],
            "ปลาหมอ": [
                "ปลาหมอมาแล้ว! มีอาการยังไงบอกหมอนะจ๊ะ",
                "ตัวเล็กแต่ใจใหญ่ นี่แหละปลาหมอของแท้",
                "เห็นเงียบๆ ปลาหมอตัวนี้ฟาดเรียบนะครับ",
                "อย่าเอาผมไปทำแกงส้มเลย ผมยังอยากอยู่ดูโลกต่อ"
            ],
            "ไม่รู้": [
                "ปลาลึกลับ! หรือจะเป็นสายพันธุ์ใหม่จากดาวอังคาร",
                "AI มึนตึ้บ! นี่มันปลาหรือก้อนหินกันแน่",
                "เอาใหม่ๆ ขอภาพชัดๆ แบบเห็นหน้าเห็นตาหน่อย"
            ]
        };

        async function init() {
            model = await tmImage.load(URL + "model.json", URL + "metadata.json");
        }

        function showPage(p) {
            document.querySelectorAll('section').forEach(s => s.classList.remove('active'));
            document.getElementById(p).classList.add('active');
        }

        async function processImage(event) {
            if (!model) await init();
            const file = event.target.files[0];
            if (!file) return;

            const reader = new FileReader();
            const imgDisplay = document.getElementById("display-image");

            reader.onload = function(e) {
                // แสดงรูปที่ผู้ใช้เลือก
                imgDisplay.src = e.target.result;
                imgDisplay.style.display = "block";
                
                imgDisplay.onload = async function() {
                    const prediction = await model.predict(imgDisplay);
                    let top = prediction.reduce((a, b) => (a.probability > b.probability) ? a : b);

                    let name = "";
                    let list = [];

                    // เปลี่ยนจาก Class 1/2 เป็นภาษาไทยล้วน
                    if(top.className === "Class 1") { 
                        name = "ปลานิล";
                        list = jokes["ปลานิล"];
                    } else if(top.className === "Class 2") {
                        name = "ปลาหมอ";
                        list = jokes["ปลาหมอ"];
                    } else {
                        name = "ตัวอะไรเนี่ย?";
                        list = jokes["ไม่รู้"];
                    }

                    const joke = list[Math.floor(Math.random() * list.length)];

                    document.getElementById("result-box").style.display = "block";
                    document.getElementById("fish-name").innerText = name;
                    document.getElementById("joke-text").innerText = "คำทำนาย: " + joke;
                    
                    // เลื่อนหน้าจอลงมาดูผลลัพธ์
                    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });
                };
            };
            reader.readAsDataURL(file);
        }

        init();
    </script>
</body>
</html>
