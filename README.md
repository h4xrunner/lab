# lab
class TransactionLogger {
    private static TransactionLogger INSTANCE = null;
    private List<String> log;

    private TransactionLogger() {
        log = new CopyOnWriteArrayList<>();
    }

    public static TransactionLogger getInstance() {
        if (INSTANCE == null) {
            synchronized (TransactionLogger.class) {
                if (INSTANCE == null) {
                    INSTANCE = new TransactionLogger();
                }
            }
        }
        return INSTANCE;
    }

    public void log(String entry) {
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        log.add(timestamp + " " + entry);
    }

    public List<String> getLog() {
        return Collections.unmodifiableList(log);
    }
}

// Custom Exceptions
class MaterialNotAvailableException extends Exception {
    public MaterialNotAvailableException(String msg) {
        super(msg);
    }
}

class MaxBorrowLimitExceededException extends Exception {
    public MaxBorrowLimitExceededException(String msg) {
        super(msg);
    }
}

class LargeFileException extends Exception {
    public LargeFileException(String msg) {
        super(msg);
    }
}

class DataFormatException extends Exception {
    public DataFormatException(String msg) {
        super(msg);
    }
}

// 2. Abstraction & Encapsulation
abstract class Material {
    private String id;
    private String title;
    private boolean isAvailable;
    protected TransactionLogger logger = TransactionLogger.getInstance();

    public Material(String id, String title) {
        this.id = id;
        this.title = title;
        this.isAvailable = true;
    }

    public String getId() { 
        return id; 
    }
    
    public String getTitle() { 
        return title; 
    }
    
    public boolean isAvailable() { 
        return isAvailable; 
    }
    
    protected void setAvailable(boolean available) { 
        this.isAvailable = available; 
    }

    public void borrow() throws MaterialNotAvailableException {
        if (!isAvailable) throw new MaterialNotAvailableException("Material " + id + " is not available");
        isAvailable = false;
        logger.log("BORROW: " + id);
    }

    public void returnItem() {
        if (isAvailable) throw new IllegalStateException("Material " + id + " is not borrowed");
        isAvailable = true;
        logger.log("RETURN: " + id);
    }

    public abstract void printDetails();
}

// 3. Downloadable Interface
interface Downloadable {
    void download() throws IllegalStateException, LargeFileException;
}

// 4. Inheritance & Polymorphism
class Book extends Material {
    private String author;
    private int pageCount;

    public Book(String id, String title, String author, int pageCount) {
        super(id, title);
        this.author = author;
        this.pageCount = pageCount;
    }

    public String getAuthor() { 
        return author; 
    }
    
    public int getPageCount() { 
        return pageCount; 
    }

    @Override
    public void borrow() throws MaterialNotAvailableException {
        super.borrow();
        if (pageCount > 300) {
            logger.log("EXTRA LOAN DAYS: " + getId() + " days=2");
        }
    }

    @Override
    public void returnItem() {
        super.returnItem();
        int reportDays = pageCount / 100;
        logger.log("READING REPORT: " + getId() + " days=" + reportDays);
    }

    @Override
    public void printDetails() {
        System.out.println("Book[id=" + getId() + ", title=" + getTitle() + 
                          ", author=" + author + ", pages=" + pageCount + 
                          ", available=" + isAvailable() + "]");
    }
}

class Magazine extends Material {
    private int issueNumber;

    public Magazine(String id, String title, int issueNumber) {
        super(id, title);
        this.issueNumber = issueNumber;
    }

    public int getIssueNumber() { 
        return issueNumber; 
    }

    @Override
    public void borrow() throws MaterialNotAvailableException {
        super.borrow();
        if (issueNumber % 2 == 0) {
            logger.log("COUPON: " + getId() + " discount=10%");
        }
    }

    @Override
    public void returnItem() {
        super.returnItem();
    }

    @Override
    public void printDetails() {
        System.out.println("Magazine[id=" + getId() + ", title=" + getTitle() + 
                          ", issue=" + issueNumber + ", available=" + isAvailable() + "]");
    }
}

class EBook extends Material implements Downloadable {
    private double fileSizeMB;
    private int downloadCount;

    public EBook(String id, String title, double fileSizeMB) {
        super(id, title);
        this.fileSizeMB = fileSizeMB;
        this.downloadCount = 0;
    }

    public EBook(String id, String title, double fileSizeMB, int downloadCount) {
        super(id, title);
        this.fileSizeMB = fileSizeMB;
        this.downloadCount = downloadCount;
    }

    public double getFileSizeMB() { 
        return fileSizeMB; 
    }
    
    public int getDownloadCount() { 
        return downloadCount; 
    }

    @Override
    public void borrow() throws MaterialNotAvailableException {
        super.borrow();
        try {
            download();
        } catch (Exception e) {
            System.err.println("Auto-download error: " + e.getMessage());
        }
    }

    @Override
    public void returnItem() {
        super.returnItem();
    }

    @Override
    public void download() throws IllegalStateException, LargeFileException {
        if (!isAvailable()) {
            throw new IllegalStateException("Item " + getId() + " not available");
        }
        downloadCount++;
        if (fileSizeMB > 100) {
            throw new LargeFileException("File size too large: " + fileSizeMB);
        }
        logger.log("DOWNLOAD: " + getId() + " size=" + fileSizeMB + "MB");
    }

    @Override
    public void printDetails() {
        System.out.println("EBook[id=" + getId() + ", title=" + getTitle() + 
                          ", sizeMB=" + fileSizeMB + ", downloads=" + downloadCount + 
                          ", available=" + isAvailable() + "]");
    }
}

// 5. Member Management & Exception Handling
class Member {
    private String memberId;
    private String name;
    private List<Material> borrowedMaterials = new ArrayList<>();
    private Map<Material, LocalDate> dueDates = new HashMap<>();
    private TransactionLogger logger = TransactionLogger.getInstance();
    public static final int MAX_BORROW = 5;

    public Member(String memberId, String name) {
        this.memberId = memberId;
        this.name = name;
    }

    public String getMemberId() { 
        return memberId; 
    }
    
    public String getName() { 
        return name; 
    }

    public void borrow(Material m) throws MaterialNotAvailableException, MaxBorrowLimitExceededException {
        if (borrowedMaterials.size() >= MAX_BORROW) {
            throw new MaxBorrowLimitExceededException("Member " + memberId + " reached max borrow limit");
        }
        
        m.borrow();
        borrowedMaterials.add(m);
        
        int loanDays = computeLoanDays(m);
        LocalDate due = LocalDate.now().plusDays(loanDays);
        dueDates.put(m, due);
        
        logger.log("DUE DATE: member=" + memberId + " material=" + m.getId() + " date=" + due);
    }

    public void returnItem(Material m) {
        if (!borrowedMaterials.contains(m)) {
            throw new IllegalArgumentException("Material " + m.getId() + " was not borrowed by this member");
        }
        
        LocalDate due = dueDates.get(m);
        if (LocalDate.now().isAfter(due)) {
            long daysLate = ChronoUnit.DAYS.between(due, LocalDate.now());
            double fee = daysLate * lateFee(m);
            logger.log("LATE FEE: member=" + memberId + " fee=" + fee);
        }
        
        m.returnItem();
        borrowedMaterials.remove(m);
        dueDates.remove(m);
    }

    private int computeLoanDays(Material m) {
        if (m instanceof Book) {
            Book book = (Book) m;
            int days = 14 + book.getPageCount() / 100;
            if (book.getPageCount() > 300) {
                days += 2;
            }
            return days;
        } else if (m instanceof Magazine) {
            return 7;
        } else if (m instanceof EBook) {
            return 30;
        }
        throw new IllegalArgumentException("Unknown material type");
    }

    private double lateFee(Material m) {
        if (m instanceof Book) {
            return 1.5;
        } else if (m instanceof Magazine) {
            return 1.0;
        } else if (m instanceof EBook) {
            return 0.5;
        }
        throw new IllegalArgumentException("Unknown material type");
    }
}

// 6. In-Memory Serialization & Deserialization
class LibraryCatalog {
    private Map<String, Material> materialsById = new HashMap<>();
    private Map<String, Member> membersById = new HashMap<>();

    public void addMaterial(Material m) {
        if (materialsById.containsKey(m.getId()))
            throw new IllegalArgumentException("Duplicate material id");
        materialsById.put(m.getId(), m);
    }

    public void removeMaterial(String id) {
        Material m = materialsById.get(id);
        if (m == null) throw new IllegalArgumentException("Material does not exist");
        if (!m.isAvailable()) throw new IllegalStateException("Material is currently borrowed");
        materialsById.remove(id);
    }

    public void registerMember(Member m) {
        if (membersById.containsKey(m.getMemberId()))
            throw new IllegalArgumentException("Duplicate member id");
        membersById.put(m.getMemberId(), m);
    }

    public List<Material> searchByTitle(String keyword) {
        List<Material> results = new ArrayList<>();
        for (Material m : materialsById.values()) {
            if (m.getTitle().toLowerCase().contains(keyword.toLowerCase())) {
                results.add(m);
            }
        }
        return results;
    }

    public void printAllDetails() {
        for (Material m : materialsById.values()) {
            m.printDetails();
        }
    }

    public List<String> serialize() {
        List<String> lines = new ArrayList<>();
        lines.add("#Members");
        for (Member m : membersById.values()) {
            lines.add(m.getMemberId() + "," + m.getName());
        }
        lines.add("#Materials");
        for (Material m : materialsById.values()) {
            if (m instanceof Book) {
                Book b = (Book) m;
                lines.add("Book," + b.getId() + "," + b.getTitle() + "," + 
                         b.isAvailable() + "," + b.getAuthor() + "," + b.getPageCount());
            } else if (m instanceof Magazine) {
                Magazine mg = (Magazine) m;
                lines.add("Magazine," + mg.getId() + "," + mg.getTitle() + "," + 
                         mg.isAvailable() + "," + mg.getIssueNumber());
            } else if (m instanceof EBook) {
                EBook eb = (EBook) m;
                lines.add("EBook," + eb.getId() + "," + eb.getTitle() + "," + 
                         eb.isAvailable() + "," + eb.getFileSizeMB() + "," + eb.getDownloadCount());
            }
        }
        return lines;
    }

    public void deserialize(List<String> lines) throws DataFormatException {
        materialsById.clear();
        membersById.clear();
        boolean inMaterials = false;
        for (String line : lines) {
            if (line.equals("#Members")) continue;
            if (line.equals("#Materials")) { inMaterials = true; continue; }
            String[] parts = line.split(",", -1);
            try {
                if (!inMaterials) {
                    if (parts.length != 2) throw new DataFormatException("Invalid member line: " + line);
                    registerMember(new Member(parts[0], parts[1]));
                } else {
                    switch (parts[0]) {
                        case "Book": {
                            if (parts.length != 6) throw new DataFormatException("Invalid book line: " + line);
                            Book b = new Book(parts[1], parts[2], parts[4], Integer.parseInt(parts[5]));
                            b.setAvailable(Boolean.parseBoolean(parts[3]));
                            addMaterial(b);
                            break;
                        }
                        case "Magazine": {
                            if (parts.length != 5) throw new DataFormatException("Invalid magazine line: " + line);
                            Magazine mg = new Magazine(parts[1], parts[2], Integer.parseInt(parts[4]));
                            mg.setAvailable(Boolean.parseBoolean(parts[3]));
                            addMaterial(mg);
                            break;
                        }
                        case "EBook": {
                            if (parts.length != 6) throw new DataFormatException("Invalid ebook line: " + line);
                            EBook eb = new EBook(parts[1], parts[2],
                                Double.parseDouble(parts[4]), Integer.parseInt(parts[5]));
                            eb.setAvailable(Boolean.parseBoolean(parts[3]));
                            addMaterial(eb);
                            break;
                        }
                        default:
                            throw new DataFormatException("Unknown material type: " + parts[0]);
                    }
                }
            } catch (NumberFormatException e) {
                throw new DataFormatException("Invalid number in line: " + line);
            }
        }
    }

    public List<String> getTransactionLog() {
        return TransactionLogger.getInstance().getLog();
    }
}

// 7. Testing in main
public class LibrarySystemTest {
    public static void main(String[] args) {
        LibraryCatalog catalog = new LibraryCatalog();
        Member alice = new Member("M1", "Alice");
        catalog.registerMember(alice);

        Book book = new Book("B1", "Effective Java", "Joshua Bloch", 416);
        Magazine mag = new Magazine("M2", "Time", 42);
        EBook ebook = new EBook("E1", "Java Concurrency", 50.0);

        catalog.addMaterial(book);
        catalog.addMaterial(mag);
        catalog.addMaterial(ebook);

        System.out.println("=== All Materials ===");
        catalog.printAllDetails();

        try {
            alice.borrow(book);
            alice.borrow(mag);
            alice.borrow(ebook);
        } catch (Exception e) {
            System.err.println("Error borrowing: " + e.getMessage());
        }

        // Explicit download test
        try {
            ebook.download();
        } catch (Exception e) {
            System.err.println("Download error: " + e.getMessage());
        }

        // Return items
        alice.returnItem(book);
        alice.returnItem(mag);
        alice.returnItem(ebook);

        // Test exceptions
        try {
            book.returnItem();
        } catch (Exception e) {
            System.err.println("Expected error: " + e.getMessage());
        }

        try {
            EBook largeEbook = new EBook("E2", "Large File", 200.0);
            catalog.addMaterial(largeEbook);
            largeEbook.download();
        } catch (Exception e) {
            System.err.println("Expected LargeFileException: " + e.getMessage());
        }

        try {
            // Exceed max borrow limit
            for (int i = 0; i < 6; i++) {
                Book b = new Book("BX" + i, "Title" + i, "Author", 100);
                catalog.addMaterial(b);
                alice.borrow(b);
            }
        } catch (Exception e) {
            System.err.println("Expected MaxBorrowLimitExceededException: " + e.getMessage());
        }

        // Serialization round-trip
        List<String> data = catalog.serialize();
        LibraryCatalog newCatalog = new LibraryCatalog();
        try {
            newCatalog.deserialize(data);
            System.out.println("=== Deserialized Materials ===");
            newCatalog.printAllDetails();
        } catch (Exception e) {
            System.err.println("Deserialize error: " + e.getMessage());
        }

        // Transaction log
        System.out.println("=== Transaction Log ===");
        for (String entry : catalog.getTransactionLog()) {
            System.out.println(entry);
        }
    }
}
