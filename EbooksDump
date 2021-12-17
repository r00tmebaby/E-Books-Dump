/*************************************************************************************
 *
 * Class EbooksDump -  ebooks.com books dumper
     Researches

     <--- API endpoints and structure ---->

      JSON Book Info Endpoints:
          https://www.ebooks.com/api/book/?bookId=118964&countryCode=UK  -> The Lowest Value
          https://www.ebooks.com/api/book/?bookId=2678800&countryCode=UK -> The Highest Value

      JSON Author Endpoints:
          https://www.ebooks.com/api/search/author/?authorId=20&pageNumber=1&countryCode=UK -> The Lowest Value
          https://www.ebooks.com/api/search/author/?authorId=123559&pageNumber=1&countryCode=UK -> The Highest Value

      Authors JSON objects in interest
          books -> list of books
          title -> Title of book
          image_url -> logo/image
          authors -> list of authors will select only the first authors[0].author_name
              author_name
          publisher -> publisher
          publication_year -> year of publication
          price -> String with locale currency sign
          desktop_short_description -> Long Description
          mobile_short_description -> Short Description

 *****************************************************************************************/
import java.io.*;
import java.net.HttpURLConnection;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import org.json.*;
import java.util.ArrayList;
import java.util.Objects;


public class EbooksDump {

        /**
         * Class Config contains all configurations for the module
         */
        public static class Config {
            public static String mainPAth         = "src/main/resources/";
            public static Boolean debugging       = true         ;   // Will display messages if true

            public static String categoriesSQL    = "data.sql"   ;  // Configure whether to use separate file for categories table
            public static String authorsSQL       = "data.sql"   ;  // Configure whether to use separate file for authors table
            public static String booksSQL         = "data.sql"   ;  // Configure whether to use separate file for books table
            public static String imagesSQL        = "data.sql"   ;  // Configure whether to use separate file for images table

            /** Note that pageEnd * categoryEnd = Number of requests (You may get blocked if that value is too high) */
            public static int pageEnd             = 10           ;   // How many books per category to iterate. Maximum 2000
            public static int categoryEnd         = 100          ;   // How many categories to iterate. Maximum 400

        }

        public static void main(String[] args) throws JSONException, IOException {

        String authors_query_string = "Insert into authors (id, full_name) values (%d, '%s');";
        String books_query_string = "Insert into books "
                + "(description, price, title, year, author_id, category_id, image_id) values "
                + "('%s', %.2f,'%s', %d, %d, %d, %d);";
        String images_query_string = "Insert into images (id, url) values (%d, '%s');";
        String categories_query_string = "Insert into categories (id, category) values (%d, '%s');";

        ArrayList<Object>   categories    = new ArrayList<>();
        ArrayList<Object>   authors       = new ArrayList<>();
        ArrayList<Object>   images        = new ArrayList<>();
        ArrayList<Book>     books         = new ArrayList<>();


        for (int catId = 1; catId <= Config.categoryEnd; catId++) {
            for (int page = 1; page <= Config.pageEnd; page++) {
                JSONObject data = get_json(catId, page);
                JSONArray booksJson = data.getJSONArray("books");
                if (booksJson.length() == 0) {
                    continue;
                }

                String category = data.getJSONArray("pages")
                        .getJSONObject(0)
                        .getString("search_url")
                        .replace("/en-gb/subjects/", "")
                        .split("/")[0].split("-")[0]
                        .toUpperCase();

                if (!categories.contains(category)) {
                    categories.add(category);
                }
                for (int book_index = 0; book_index < booksJson.length(); book_index++) {

                    boolean available = booksJson.getJSONObject(book_index).getBoolean("is_available");

                    if (!available) {
                        continue;
                    }
                    String title = booksJson.getJSONObject(book_index)
                            .getString("title")
                            .replaceAll("'", " ")
                            .replaceAll("’", " ");
                    String description = booksJson.getJSONObject(book_index)
                            .getString("desktop_short_description")
                            .replaceAll("'", " ")
                            .replaceAll("’", " ");
                    Double price = Math.round(booksJson.getJSONObject(book_index)
                            .getDouble("price_number") * 100) / 100D;
                    String image = booksJson.getJSONObject(book_index)
                            .getString("image_url")
                            .replaceAll("-sml-1", "");
                    Integer year = booksJson
                            .getJSONObject(book_index)
                            .getInt("publication_year");
                    JSONArray _author = booksJson.getJSONObject(book_index).getJSONArray("authors");

                    // Fix author object appears to be empty sometimes
                    if(_author.length() == 0){
                        continue;
                    }

                    String author = _author
                            .getJSONObject(0)
                            .getString("author_name")
                            .replaceAll("'", " ")
                            .replaceAll("’", " ");


                    if (!authors.contains(author)) {
                        authors.add(author);
                    }
                    if (!images.contains(image)) {
                        images.add(image);
                    }
                    Book newBook = new Book(title, description, price, year, image, category, author);

                    if(isAdded(books, newBook)){
                       continue;
                    }

                    books.add(newBook);

                    if (Config.debugging) {
                        System.out.println(
                                        "| Categories:" + categories.size()+
                                        "| Images|"     + images.size()+
                                        "| Authors|"    + authors.size()+
                                        "| Books|"      + books.size()+
                                        "| Current Endpoint: https://www.ebooks.com/api/search/subject/?" +
                                        "subjectId="    + catId +
                                        "&pageNumber="  + page +
                                        "&countryCode=UK"
                        );
                    }
                }
            }
        }
            for (int i = 0; i < categories.size(); i++) {
                queryGen(Config.mainPAth + Config.categoriesSQL, String.format(categories_query_string, i+1, categories.get(i)));
            }
            for (int i = 0; i < authors.size(); i++) {
                queryGen(Config.mainPAth + Config.authorsSQL, String.format(authors_query_string, i+1, authors.get(i)));
            }
            for (int i = 0; i < images.size(); i++) {
                queryGen(Config.mainPAth + Config.imagesSQL, String.format(images_query_string, i+1, images.get(i)));
            }


            for (int i = 0; i < books.size(); i++) {
                Book book = books.get(i);

                queryGen(Config.mainPAth + Config.booksSQL, String.format(books_query_string,
                        book.description,
                        book.price,
                        book.title,
                        book.year,
                        authors.indexOf(book.author)+1,
                        categories.indexOf(book.category)+1,
                        images.indexOf(book.image)+1
                        )
                );
            }
    }



    /**
     * Method queryGen creates a sql query file with all queries
     * @param filename Filename in which we will append data  (sql file)
     * @param text Text that will be appended (query)
     */
    public static void queryGen(String filename, String text) {
        try {
            FileWriter myWriter = new FileWriter(filename,true);
            myWriter.write(text + "\n");
            myWriter.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    /**
     *  Class Book will hold our Book object
     */
    public static class Book  {

        private String title;
        private String description;
        private Double price;
        private Integer year;
        private String image;
        private String category;
        private String author;


        public Book(String title, String description, double price, Integer year, String image, String category, String author) {
            this.title = title;
            this.description = description;
            this.price = price;
            this.year = year;
            this.image = image;
            this.category = category;
            this.author = author;
        }
    }

    /**
     * Checks for similarities
     * @param books Books arraylist
     * @param newBook Book object
     * @return True if the book is already added and false otherwise
     */
    public static Boolean isAdded(ArrayList<Book> books, Book newBook){
        for (int i = 0; i < books.size(); i++) {
            if(Objects.equals(books.get(i).description, newBook.description) || Objects.equals(books.get(i).title, newBook.title)){
                return true;
            }
        }
        return false;
    }
    /**
     * ************ Method get_json ******************
     *
     * Send a requests to ebooks.com website and reads the JSON Object and returns it
     * Returns an empty book JSOnArray if the status code is not 200(OK)
     * So the iteration will continue without an error
     *
     * @param catId
     * @param pageNumber
     * @return
     * @throws JSONException
     * @throws IOException
     */
    public static JSONObject get_json(Integer catId, Integer pageNumber) throws JSONException, IOException {

        String url = "https://www.ebooks.com/api/search/subject/?subjectId=" + catId + "&pageNumber=" + pageNumber + "&countryCode=UK";

        HttpURLConnection connection = (HttpURLConnection) (new URL(url).openConnection());
        connection.setRequestProperty("User-Agent", "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.11 (KHTML, like Gecko) Chrome/23.0.1271.95 Safari/537.11");
        connection.connect();

        if (connection.getResponseCode() == 200) {
            BufferedReader buffer = new BufferedReader(new InputStreamReader(connection.getInputStream(), StandardCharsets.UTF_8));
            StringBuilder bytes = new StringBuilder();
            String line;
            while ((line = buffer.readLine()) != null) {
                bytes.append(line);
            }
            return new JSONObject(bytes.toString());
        } else {
            return new JSONObject("{\n" +
                    "  \"books\": [],\n" +
                    "}");
        }
    }
}


