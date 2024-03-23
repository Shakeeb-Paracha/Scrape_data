import scrapy
from pathlib import Path
from pymongo import MongoClient
import datetime

client = MongoClient("mongodb+srv://shakeeb_paracha:smp$2020@cluster0.bvxqgmf.mongodb.net/")
db = client.scripy_database  # Corrected database name

def insertToDb(page, title, rating, image, price, in_stock):
    collection = db[page]
    doc = {
        "title": title,  # Extract text from Selector object
        "rating": rating,
        "image": image,
        "price": price,
        "in_stock": in_stock,
        "date": datetime.datetime.utcnow()
    }
    inserted = collection.insert_one(doc)
    return inserted.inserted_id

class BooksSpider(scrapy.Spider):
    name = "books"
    allowed_domains = ["toscrape.com"]
    start_urls = ["https://toscrape.com"]

    def start_requests(self):
        urls = [
            "https://books.toscrape.com/catalogue/category/books/travel_2/index.html",
            "https://books.toscrape.com/catalogue/category/books/mystery_3/index.html",
        ]
        for url in urls:
            yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response):
        page = response.url.split("/")[-2]
        filename = f"books-{page}.html"
        Path(filename).write_bytes(response.body)
        self.log(f"Saved file {filename}")
        
        cards = response.css(".product_pod")
        for card in cards:
            title = card.css("h3 > a::text").get()  # Extracting text from Selector object
            image = card.css(".image_container img::attr(src)").get().replace("../../../../media","https://books.toscrape.com/media")
            price = card.css(".price_color::text").get().replace("£", "")
            rating = card.css(".star-rating").attrib["class"].split(" ")[1]
            in_stock = card.css(".instock::text").get() 
            if in_stock == "In stock":
                in_stock = 0
            else:
                in_stock = 1
            
            insertToDb(page, title, rating, image, price, in_stock)
