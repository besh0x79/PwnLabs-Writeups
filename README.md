# ☁️ Cloud Pentest Writeups

> A collection of hands-on writeups from my journey through cloud penetration testing labs.

---

## 👾 About

Hey pwners! This repo documents my progress through cloud security labs with a focus on real-world attack techniques, enumeration methods, and exploitation paths.

All writeups are based on labs from **[PwnLabs](https://pwnlabs.io)** — a platform dedicated to practical cloud penetration testing.

---

## 📂 Structure

```
cloud-pentest-writeups/
│
├── pwnlabs/
│   ├── aws/
│   │   └── AWS_S3_Enumeration_Basics.md
│   └── ...
└── README.md
```

---

## ☁️ AWS

| Lab | Platform | Topic | Difficulty |
|-----|----------|-------|------------|
| [S3 Enumeration Basics](./pwnlabs/aws/AWS_S3_Enumeration_Basics.md) | PwnLabs | S3 Enumeration, Credential Exposure | 🟢 Easy |

---

## 🛠️ Tools Used

- AWS CLI
- `aws sts get-caller-identity`
- `aws s3 ls / cp`

---

## ⚠️ Disclaimer

All content in this repo is for **educational purposes only**.
These writeups are based on **legal lab environments**.
Do not use any of these techniques on systems you don't own or have explicit permission to test.

---

## 🔗 Connect

- Platform: [PwnLabs](https://pwnlabs.io)

---

*Stay curious, keep enumerating!* 🚀
