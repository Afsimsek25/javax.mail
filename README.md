
<body>
    <a href="https://www.linkedin.com/in/afsimsek/" target="_blank" class="ad-card">
        <img src="https://raw.githubusercontent.com/Afsimsek25/javax.mail/main/adcard.gif" alt="Ad Card Image">
    </a>
</body>

## Javax.Mail ile Mail Operasyonları Otomasyonu
Bu Class, Javax.Mail kütüphanesini kullanarak e-posta kutusuna bağlanma, belirli anahtar kelimeleri içeren e-postaları okuma ve silme işlemlerini gerçekleştirir.


## Teknolojiler
- **Java**
- **Javax.Mail**
- **Selenium 4**

## Dependencies
```xml
<dependency>
    <groupId>com.sun.mail</groupId>
    <artifactId>javax.mail</artifactId>
    <version>1.6.2</version>
</dependency>
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>4.27.0</version>
</dependency>
```


## Mail Sağlayıcısının Hazırlığı.
##### Örnek olarak Yandex Mail kullanılmıştır. Diğer sağlayıcılar için benzer yapılandırmalar uygulanabilir.
	1.	Yandex web arayüzüne giriş yapın.
	2.	Sağ üst köşeden “Ayarlar” sekmesine tıklayın.
	3.	“E-posta istemcileri” bölümünü bulun ve IMAP desteğini etkinleştirin.
	4.	“Güvenlik” sekmesine giderek uygulama şifresi oluşturun.

## Development
####  Main.java:
```java
public static void main(String[] args) {
        MailOperations mailOperations = new MailOperations();

        /**
         * @param userName  IMAP Yetkilendirilmesi yapılmış Mail adresiniz.
         * @param password  mailinizin uygulama erişim şifresi (Güvenlikten eklenecek).
         * @param searchedWord  Hedef mailde arayacağınız referans kelime.
         * @param filePath  Mailin export edileceği path
         */
        mailOperations.getMails("mailiniz@yandex.com","Uygulama Erişim Şifreniz","Aradığınız Kelime","/Users/afsim/Downloads/mail2.html");

        WebDriver driver = new ChromeDriver();

        /**
         * @param url   Export edilen html dosyasının path'i. (navigate olabilmek için başına "file://" eklemeniz lazım)
         */
        driver.navigate().to("file:///Users/afsim/Downloads/mail2.html");

```



#### MailOperations.java  Methodları


```java
    private Properties properties = null;
    private Session session = null;
    private Store store = null;
    private Folder inbox = null;
    private Message[] messages;
    private String filePath;
```

```java
    /**
     * Başlık veya içeriğinde aranan kelimenin bulunduğu Mail'i arar, içeriğini alır ve verilen Path'e HTML file olarak yazar.
     *
     * @param userName    Mail hesabının kullanıcı adı.
     * @param password    Mail 3. parti Uygulama şifresi.
     * @param searchedWord the keyword to search for in emails.
     * @param filePath    the file path where the result will be saved.
     */
    public void getMails(String userName, String password, String searchedWord, String filePath) {
        this.filePath = filePath;
        getConnection(userName, password);
        try {
            readMails(searchedWord);
        } finally {
            closeSession();
        }
    }
```

```java
    /**
     * Bu method IMAP protokolü ile yandex e-posta sunucusuna bir bağlantı kurar ve INBOX klasörünü açar.
     *
     * @param userName Mail hesabının kullanıcı adı.
     * @param password Mail 3. parti Uygulama şifresi.
     */
    private void getConnection(String userName, String password) {
        properties = new Properties();
        properties.setProperty("mail.host", "imap.yandex.com");
        properties.setProperty("mail.port", "995");
        properties.setProperty("mail.transport.protocol", "imaps");

        session = Session.getInstance(properties, new Authenticator() {
            @Override
            protected PasswordAuthentication getPasswordAuthentication() {
                return new PasswordAuthentication(userName, password);
            }
        });

        try {
            store = session.getStore("imaps");
            store.connect();
            inbox = store.getFolder("INBOX");
            inbox.open(Folder.READ_WRITE);
        } catch (MessagingException e) {
            System.err.println("Failed to connect to the mail server: " + e.getMessage());
        }
    }
```

```java
    /**
     * Belirtilen anahtar kelimeyi içeren e-postaları bulur ve içeriği bir dosyaya kaydeder .html formatında kaydeder.
     *
     * @param searchedWord Mailler içerisinde aranacak anahtar kelime.
     */

    private void readMails(String searchedWord) {
        try {
            messages = inbox.getMessages();
            for (Message message : messages) {
                if (messageMatchesKeyword(message, searchedWord)) {
                    saveMessageToFile(message);
                }
            }
        } catch (MessagingException | IOException e) {
            System.err.println("Error reading emails: " + e.getMessage());
        }
    }
```

```java 

    /**
     * Bir mesajın konusu veya gövdesinin belirtilen anahtar kelimeyi içerip içermediğini kontrol eder.
     *
     * @param message kontrol edilecek Mail.
     * @param searchedWord aranan anahtar kelime.
     * @return anahtar kelime bulunursa true; aksi takdirde false.
     */

    private boolean messageMatchesKeyword(Message message, String searchedWord) throws MessagingException, IOException {
        String subject = message.getSubject() != null ? message.getSubject().toLowerCase() : "";
        String content = message.getContent() != null ? message.getContent().toString().toLowerCase() : "";

        return subject.contains(searchedWord.toLowerCase()) || content.contains(searchedWord.toLowerCase());
    }
```
```java 
    /**
     * Mail'in içeriğini alır.
     *
     * @param message içeriği alınmak istenen Mail.
     * @return Mesajın içeriğini bir Srting olarak döndürür veya Multipart method'una yönlendirir.
     */
    private String getMessageContent(Message message) throws MessagingException, IOException {
        Object content = message.getContent();
        if (content instanceof String) {
            return (String) content;
        } else if (content instanceof Multipart) {
            return getMultipartContent((Multipart) content);
        }
        return "";
    }
```
```java 
/**
     * Çok parçalı bir Mail'in içeriğini işler ve return eder.
     *
     * @param multipart Çok parçalı mail içeriği.
     * @return Birleştirilmiş mail parçacıkları. (HTML, String)
     */
    private String getMultipartContent(Multipart multipart) throws MessagingException, IOException {
        StringBuilder content = new StringBuilder();
        for (int i = 0; i < multipart.getCount(); i++) {
            BodyPart bodyPart = multipart.getBodyPart(i);
            Object partContent = bodyPart.getContent();
            if (partContent instanceof String) {
                content.append(partContent.toString());
            } else if (partContent instanceof Multipart) {
                content.append(getMultipartContent((Multipart) partContent));
            }
        }
        return content.toString();
    }
```
```java
    /**
     * Mail'in içeriğini verilen dosya yoluna kaydeder.
     *
     * @param message search sonucunda bulunup kaydedilmek istenen il
     */
    private void saveMessageToFile(Message message) throws IOException, MessagingException {
        String content = getMessageContent(message);
        writeToFile(filePath, content);
    }
```
```java 
    /**
     * Mail içeriğini file'a yazar.
     *
     * @param filePath File Path
     * @param content  File'a yazılacak içerik
     */
    private void writeToFile(String filePath, String content) throws IOException {
        try (FileWriter writer = new FileWriter(filePath)) {
            writer.write(content);
        }
    }
```
```java 
    /**
     * Okuma ve kaydetme işlemi biten Mail'i siler.
     *
     * @param message Silinmek istenen Mail.
     */
    private void deleteMessage(Message message) {
        try {
            message.setFlag(Flag.DELETED, true);
            System.out.println("Deleted mail: " + message.getSubject());
        } catch (MessagingException e) {
            System.err.println("Error deleting mail: " + e.getMessage());
        }
    }
```
```java 
    /**
     * Posta kutusundaki tüm Mailleri (hem okunmuş hem de okunmamış) siler.
     *
     * @param userName Mail Adresi
     * @param password Mail 3. parti Uygulama şifresi.
     */
    public void deleteAllMail(String userName, String password) {
        getConnection(userName, password); // Bağlantı kuruluyor
        try {
            if (inbox != null && inbox.isOpen()) {
                // Silinmemiş tüm mesajları getir
                messages = inbox.search(new FlagTerm(new Flags(Flags.Flag.DELETED), false));
                System.out.println("Number of mails to delete: " + messages.length);
                for (Message message : messages) {
                    deleteMessage(message); // Mesajları işaretle ve sil
                }
            } else {
                System.out.println("Inbox folder is not open or does not exist.");
            }
        } catch (MessagingException e) {
            // Hata mesajını konsola yazdır ve uygun bir exception fırlat
            System.err.println("Error while accessing or deleting emails: " + e.getMessage());
            throw new RuntimeException("Error during mail deletion process: " + e.getMessage(), e);
        } finally {
            closeSession(); // Oturumu kapat
        }
    }
```
```java 
    /**
     * Mail' işlemleri tamamlandıktan sonra bir önceki işlemden önce mevcut Session'u sonlandırmak gerekir.
     */
    private void closeSession() {
        try {
            if (inbox != null && inbox.isOpen()) {
                inbox.close(true);
            }
            if (store != null) {
                store.close();
            }
        } catch (MessagingException e) {
            System.err.println("Error closing session: " + e.getMessage());
        }
    }
```


[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/afsimsek/)
