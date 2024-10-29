// models/Visitor.js
const mongoose = require('mongoose');

const visitorSchema = new mongoose.Schema({
    ip: { type: String, required: true, unique: true },
    visitCount: { type: Number, default: 1 },
    lastVisit: { type: Date, default: Date.now },
}, { timestamps: true });

module.exports = mongoose.model('Visitor', visitorSchema);

// index.js
const express = require('express');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const nodemailer = require('nodemailer');
require('dotenv').config();
const Visitor = require('./models/Visitor'); // Импортируйте модель

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(bodyParser.json());
app.use(express.static('public'));

// Подключение к MongoDB
mongoose.connect(process.env.MONGODB_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
})
.then(() => console.log('MongoDB connected successfully!'))
.catch(err => console.error('MongoDB connection error:', err));

// Обработчик для отслеживания посещений
app.post('/track-visitor', async (req, res) => {
    const { ip } = req.body; // Получаем IP-адрес

    try {
        // Проверяем, есть ли уже запись о посещении
        let visitor = await Visitor.findOne({ ip });

        if (visitor) {
            // Увеличиваем счетчик посещений и обновляем время последнего посещения
            visitor.visitCount++;
            visitor.lastVisit = Date.now();
            await visitor.save();

            // Проверяем, превышает ли количество посещений порог
            if (visitor.visitCount > process.env.ALERT_THRESHOLD) {
                sendEmail(visitor);
            }
        } else {
            // Создаем новую запись о посещении
            visitor = new Visitor({ ip });
            await visitor.save();
        }
        
        res.status(200).json({ message: 'Visitor tracked successfully!' });
    } catch (error) {
        res.status(500).json({ message: 'Error tracking visitor', error });
    }
});

// Функция для отправки уведомлений по электронной почте
function sendEmail(visitor) {
    const transporter = nodemailer.createTransport({
        service: 'gmail',
        auth: {
            user: process.env.EMAIL_USER,
            pass: process.env.EMAIL_PASS,
        },
    });

    const mailOptions = {
        from: process.env.EMAIL_USER,
        to: process.env.EMAIL_USER,
        subject: 'Visitor Alert',


        text: `Visitor alert! IP: ${visitor.ip} has visited the site ${visitor.visitCount} times.`,
    };

    transporter.sendMail(mailOptions, (error, info) => {
        if (error) {
            return console.log(error);
        }
        console.log('Email sent: ' + info.response);
    });
}

// Запускаем сервер
app.listen(PORT, () => {
    console.log(`Server is running on http://localhost:${PORT}`);
});

/* public/styles.css */
body {
    font-family: Arial, sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100vh;
    margin: 0;
    background-color: #f8f8f8;
}

h1 {
    color: #333;
}


        text: `Visitor alert! IP: ${visitor.ip} has visited the site ${visitor.visitCount} times.`,
    };

    transporter.sendMail(mailOptions, (error, info) => {
        if (error) {
            return console.log(error);
        }
        console.log('Email sent: ' + info.response);
    });
}

// Запускаем сервер
app.listen(PORT, () => {
    console.log(`Server is running on http://localhost:${PORT}`);
});


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Website Visitor Alert</title>
    <link rel="stylesheet" href="styles.css">
    <script defer src="app.js"></script>
</head>
<body>
    <h1>Website Visitor Tracking</h1>
    <p>Track your visitors and get alerts!</p>
    <div id="message"></div>
</body>
</html>


/* public/styles.css */
body {
    font-family: Arial, sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100vh;
    margin: 0;
    background-color: #f8f8f8;
}

h1 {
    color: #333;
}


// public/app.js
window.onload = async () => {
    const messageElement = document.getElementById('message');
    
    // Получаем IP-адрес
    try {
        const response = await fetch('https://api.ipify.org?format=json');
        const data = await response.json();
        const ip = data.ip;

        // Отправляем IP-адрес на сервер
        const trackResponse = await fetch('/track-visitor', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ ip })
        });

        if (trackResponse.ok) {
            messageElement.innerText = 'Visitor tracked successfully!';
        } else {
            const errorData = await trackResponse.json();
            messageElement.innerText = errorData.message;
        }
    } catch (error) {
        messageElement.innerText = 'Error retrieving IP address.';
    }
};
