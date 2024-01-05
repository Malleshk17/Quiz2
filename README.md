import java.io.*;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;

public class LinkShortener {
    private static final String DATA_FILE = "link_mappings.txt";
    private HashMap<String, String> shortToLongMap;
    private HashMap<String, String> longToShortMap;

    public LinkShortener() {
        shortToLongMap = new HashMap<>();
        longToShortMap = new HashMap<>();
        loadLinkMappings(); // Load existing mappings from file
    }

    public String shortenUrl(String longUrl) {
        validateLongUrl(longUrl);

        if (longToShortMap.containsKey(longUrl)) {
            return longToShortMap.get(longUrl);
        }

        String shortCode = generateShortCode(longUrl);
        while (shortToLongMap.containsKey(shortCode)) {
            shortCode = generateShortCode(longUrl);
        }

        shortToLongMap.put(shortCode, longUrl);
        longToShortMap.put(longUrl, shortCode);
        saveLinkMappings(); // Save updated mappings to file

        return shortCode;
    }

    public String expandUrl(String shortUrl) {
        validateShortUrl(shortUrl);

        return shortToLongMap.getOrDefault(shortUrl, "URL not found");
    }

    private void validateLongUrl(String longUrl) {
        if (longUrl == null || longUrl.isEmpty()) {
            throw new IllegalArgumentException("Invalid long URL");
        }
    }

    private void validateShortUrl(String shortUrl) {
        if (shortUrl == null || shortUrl.isEmpty() || !shortToLongMap.containsKey(shortUrl)) {
            throw new IllegalArgumentException("Invalid short URL");
        }
    }

    private String generateShortCode(String longUrl) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(longUrl.getBytes());
            StringBuilder shortCode = new StringBuilder();

            for (int i = 0; i < Math.min(6, hash.length); i++) {
                shortCode.append(Integer.toString((hash[i] & 0xff) + 0x100, 16).substring(1));
            }

            return shortCode.toString();
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("Error generating short code", e);
        }
    }

    private void saveLinkMappings() {
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(DATA_FILE))) {
            Map<String, String> linkMappings = new HashMap<>();
            linkMappings.putAll(shortToLongMap);
            linkMappings.putAll(longToShortMap);
            oos.writeObject(linkMappings);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void loadLinkMappings() {
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(DATA_FILE))) {
            Map<String, String> linkMappings = (Map<String, String>) ois.readObject();
            shortToLongMap.putAll(linkMappings);
            for (Map.Entry<String, String> entry : linkMappings.entrySet()) {
                longToShortMap.put(entry.getValue(), entry.getKey());
            }
        } catch (IOException | ClassNotFoundException e) {
            // Handle file not found or corrupted file
            System.out.println("No existing link mappings found.");
        }
    }

    public static void main(String[] args) {
        LinkShortener linkShortener = new LinkShortener();

        // Example usage:
        String longUrl = "https://www.example.com";
        String shortUrl = linkShortener.shortenUrl(longUrl);
        System.out.println("Shortened URL: " + shortUrl);

        String expandedUrl = linkShortener.expandUrl(shortUrl);
        System.out.println("Expanded URL: " + expandedUrl);

        // Handle invalid short URL
        try {
            linkShortener.expandUrl("invalidShortUrl");
        } catch (IllegalArgumentException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
