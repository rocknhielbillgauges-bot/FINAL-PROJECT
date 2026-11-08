using System;
using System.Globalization;

namespace WarehouseStockSystem
{
    class Program
    {
        // ---------- Configuration ----------
        const int MaxProducts = 100;
        const int MaxLog = 500;
        const int LowProductLevel = 10;    // a product below this in one warehouse is "low"
        const int LowWarehouseLevel = 50;  // a warehouse whose total units are below this is "low"

        static readonly string[] WarehouseNames = { "Warehouse A", "Warehouse B", "Warehouse C" };
        static int WarehouseCount => WarehouseNames.Length;

        // ---------- Parallel arrays: product data ----------
        static string[] codes = new string[MaxProducts];
        static string[] names = new string[MaxProducts];
        static decimal[] prices = new decimal[MaxProducts];
        static int[,] qty = new int[MaxProducts, WarehouseNames.Length]; // [product, warehouse]
        static int productCount = 0;

        // ---------- Parallel arrays: movement log ----------
        static string[] logTime = new string[MaxLog];
        static string[] logType = new string[MaxLog];
        static string[] logCode = new string[MaxLog];
        static int[] logQty = new int[MaxLog];
        static string[] logDetail = new string[MaxLog];
        static int logCount = 0;

        static void Main()
        {
            SeedSampleData();
            bool running = true;

            while (running)
            {
                ShowMenu();
                int choice = ReadInt("Enter choice: ", 0, 9);

                switch (choice)
                {
                    case 0: running = false; continue;
                    case 1: ReceiveStock(); break;
                    case 2: TransferStock(); break;
                    case 3: ReleaseStock(); break;
                    case 4: SearchProduct(); break;
                    case 5: UpdateInventory(); break;
                    case 6: LowInventory(); break;
                    case 7: StockValue(); break;
                    case 8: MovementRecords(); break;
                    case 9: InventorySummary(); break;
                }

                Console.WriteLine();
                Console.Write("Press any key to return to the menu...");
                Console.ReadKey(true);
            }

            Console.WriteLine();
            Console.WriteLine("Goodbye!");
        }

        // =====================================================
        // MENU
        // =====================================================
        static void ShowMenu()
        {
            Console.Clear();
            Console.WriteLine("=====================================");
            Console.WriteLine("  WAREHOUSE STOCK DISTRIBUTION SYSTEM");
            Console.WriteLine("=====================================");
            Console.WriteLine(" 1. Receive stock");
            Console.WriteLine(" 2. Transfer stock");
            Console.WriteLine(" 3. Release stock");
            Console.WriteLine(" 4. Search product");
            Console.WriteLine(" 5. Update inventory");
            Console.WriteLine(" 6. Low inventory");
            Console.WriteLine(" 7. Stock value per warehouse");
            Console.WriteLine(" 8. Movement records");
            Console.WriteLine(" 9. Inventory summary");
            Console.WriteLine(" 0. Exit");
            Console.WriteLine("-------------------------------------");
        }

        // =====================================================
        // 1. RECEIVE STOCK
        // =====================================================
        static void ReceiveStock()
        {
            Header("RECEIVE STOCK");
            string code = ReadText("Product code: ").ToUpper();
            int p = FindByCode(code);

            if (p < 0)
            {
                if (productCount >= MaxProducts)
                {
                    Console.WriteLine("Product list is full. Cannot add a new product.");
                    return;
                }
                Console.WriteLine("New product. Enter its details.");
                string name = ReadText("Product name: ");
                decimal price = ReadDecimal("Unit price: ", 0m);

                p = productCount++;
                codes[p] = code;
                names[p] = name;
                prices[p] = price;
            }
            else
            {
                Console.WriteLine("Existing product: " + names[p]);
            }

            int w = ChooseWarehouse("Receive into");
            int amount = ReadInt("Quantity received: ", 1, int.MaxValue);

            qty[p, w] += amount;
            AddLog("RECEIVE", codes[p], amount, "Into " + WarehouseNames[w]);
            Console.WriteLine("Received " + amount + " x " + names[p] + " into " + WarehouseNames[w] +
                              ". New quantity there: " + qty[p, w]);
        }

        // =====================================================
        // 2. TRANSFER STOCK
        // =====================================================
        static void TransferStock()
        {
            Header("TRANSFER STOCK");
            int p = AskForExistingProduct();
            if (p < 0) return;

            int from = ChooseWarehouse("Transfer FROM");
            Console.WriteLine("Available in " + WarehouseNames[from] + ": " + qty[p, from]);
            if (qty[p, from] == 0)
            {
                Console.WriteLine("Nothing to transfer from this warehouse.");
                return;
            }

            int to = ChooseWarehouse("Transfer TO");
            if (to == from)
            {
                Console.WriteLine("Source and destination must be different.");
                return;
            }

            int amount = ReadInt("Quantity to transfer: ", 1, int.MaxValue);

            // Prevent transfers that exceed available quantity
            if (amount > qty[p, from])
            {
                Console.WriteLine("Transfer refused: only " + qty[p, from] + " available in " +
                                  WarehouseNames[from] + ".");
                return;
            }

            qty[p, from] -= amount;
            qty[p, to] += amount;
            AddLog("TRANSFER", codes[p], amount, WarehouseNames[from] + " -> " + WarehouseNames[to]);
            Console.WriteLine("Transferred " + amount + " x " + names[p] + " from " +
                              WarehouseNames[from] + " to " + WarehouseNames[to] + ".");
        }

        // =====================================================
        // 3. RELEASE STOCK
        // =====================================================
        static void ReleaseStock()
        {
            Header("RELEASE STOCK FOR DISTRIBUTION");
            int p = AskForExistingProduct();
            if (p < 0) return;

            int w = ChooseWarehouse("Release FROM");
            Console.WriteLine("Available in " + WarehouseNames[w] + ": " + qty[p, w]);
            if (qty[p, w] == 0)
            {
                Console.WriteLine("Nothing to release from this warehouse.");
                return;
            }

            int amount = ReadInt("Quantity to release: ", 1, int.MaxValue);

            if (amount > qty[p, w])
            {
                Console.WriteLine("Release refused: only " + qty[p, w] + " available in " +
                                  WarehouseNames[w] + ".");
                return;
            }

            string destination = ReadText("Destination / recipient: ");

            qty[p, w] -= amount;
            AddLog("RELEASE", codes[p], amount, "From " + WarehouseNames[w] + " to " + destination);
            Console.WriteLine("Released " + amount + " x " + names[p] + " from " + WarehouseNames[w] + ".");
        }

        // =====================================================
        // 4. SEARCH PRODUCT
        // =====================================================
        static void SearchProduct()
        {
            Header("SEARCH PRODUCT");
            string term = ReadText("Enter product code or name (partial allowed): ").ToLower();
            int found = 0;

            for (int i = 0; i < productCount; i++)
            {
                if (codes[i].ToLower() == term || names[i].ToLower().Contains(term))
                {
                    if (found == 0) PrintProductHeader();
                    PrintProductRow(i);
                    found++;
                }
            }

            if (found == 0) Console.WriteLine("No matching product found.");
            else Console.WriteLine(found + " product(s) found.");
        }

        // =====================================================
        // 5. UPDATE INVENTORY
        // =====================================================
        static void UpdateInventory()
        {
            Header("UPDATE INVENTORY");
            int p = AskForExistingProduct();
            if (p < 0) return;

            Console.WriteLine("What would you like to update?");
            Console.WriteLine(" 1. Quantity (set a warehouse count)");
            Console.WriteLine(" 2. Price");
            Console.WriteLine(" 3. Name");
            int field = ReadInt("Choice: ", 1, 3);

            if (field == 1)
            {
                int w = ChooseWarehouse("Warehouse to adjust");
                Console.WriteLine("Current quantity: " + qty[p, w]);
                int newQty = ReadInt("New quantity: ", 0, int.MaxValue);
                int diff = newQty - qty[p, w];
                qty[p, w] = newQty;
                AddLog("UPDATE-QTY", codes[p], Math.Abs(diff),
                       WarehouseNames[w] + " adjusted " + (diff >= 0 ? "+" : "-") + Math.Abs(diff) +
                       " (now " + newQty + ")");
                Console.WriteLine("Quantity updated.");
            }
            else if (field == 2)
            {
                Console.WriteLine("Current price: " + prices[p].ToString("N2"));
                decimal newPrice = ReadDecimal("New unit price: ", 0m);
                string detail = "Price " + prices[p].ToString("N2") + " -> " + newPrice.ToString("N2");
                prices[p] = newPrice;
                AddLog("UPDATE-PRICE", codes[p], 0, detail);
                Console.WriteLine("Price updated.");
            }
            else
            {
                Console.WriteLine("Current name: " + names[p]);
                string newName = ReadText("New name: ");
                string detail = "Name '" + names[p] + "' -> '" + newName + "'";
                names[p] = newName;
                AddLog("UPDATE-NAME", codes[p], 0, detail);
                Console.WriteLine("Name updated.");
            }
        }

        // =====================================================
        // 6. LOW INVENTORY
        // =====================================================
        static void LowInventory()
        {
            Header("LOW INVENTORY REPORT");
            Console.WriteLine("Warehouse flagged if total units < " + LowWarehouseLevel);
            Console.WriteLine("Product flagged if units in a warehouse < " + LowProductLevel);
            Console.WriteLine();

            bool anyFlag = false;

            for (int w = 0; w < WarehouseCount; w++)
            {
                int total = WarehouseUnits(w);
                bool lowWarehouse = total < LowWarehouseLevel;

                Console.WriteLine(WarehouseNames[w] + " - total units: " + total +
                                  (lowWarehouse ? "  ** LOW WAREHOUSE **" : ""));
                if (lowWarehouse) anyFlag = true;

                for (int p = 0; p < productCount; p++)
                {
                    if (qty[p, w] < LowProductLevel)
                    {
                        Console.WriteLine("    low: " + codes[p].PadRight(8) + names[p].PadRight(22) +
                                          qty[p, w] + " left");
                        anyFlag = true;
                    }
                }
                Console.WriteLine();
            }

            if (!anyFlag) Console.WriteLine("All warehouses are adequately stocked.");
        }

        // =====================================================
        // 7. STOCK VALUE PER WAREHOUSE
        // =====================================================
        static void StockValue()
        {
            Header("STOCK VALUE PER WAREHOUSE");
            decimal grand = 0m;

            Console.WriteLine("Warehouse".PadRight(16) + "Units".PadLeft(8) + "Value".PadLeft(18));
            Console.WriteLine(new string('-', 42));

            for (int w = 0; w < WarehouseCount; w++)
            {
                decimal value = WarehouseValue(w);
                grand += value;
                Console.WriteLine(WarehouseNames[w].PadRight(16) +
                                  WarehouseUnits(w).ToString().PadLeft(8) +
                                  value.ToString("N2").PadLeft(18));
            }

            Console.WriteLine(new string('-', 42));
            Console.WriteLine("TOTAL".PadRight(16) + TotalUnits().ToString().PadLeft(8) +
                              grand.ToString("N2").PadLeft(18));
        }

        // =====================================================
        // 8. MOVEMENT RECORDS
        // =====================================================
        static void MovementRecords()
        {
            Header("STOCK MOVEMENT RECORDS");
            if (logCount == 0)
            {
                Console.WriteLine("No movements recorded yet.");
                return;
            }

            Console.WriteLine("#".PadRight(4) + "Time".PadRight(18) + "Type".PadRight(14) +
                              "Code".PadRight(8) + "Qty".PadLeft(6) + "  Details");
            Console.WriteLine(new string('-', 80));

            for (int i = 0; i < logCount; i++)
            {
                Console.WriteLine((i + 1).ToString().PadRight(4) + logTime[i].PadRight(18) +
                                  logType[i].PadRight(14) + logCode[i].PadRight(8) +
                                  logQty[i].ToString().PadLeft(6) + "  " + logDetail[i]);
            }
        }

        // =====================================================
        // 9. INVENTORY SUMMARY
        // =====================================================
        static void InventorySummary()
        {
            Header("OVERALL INVENTORY SUMMARY");
            if (productCount == 0)
            {
                Console.WriteLine("No products in the system.");
                return;
            }

            PrintProductHeader();
            for (int i = 0; i < productCount; i++) PrintProductRow(i);

            Console.WriteLine();
            Console.WriteLine("Products on record : " + productCount);
            Console.WriteLine("Total units        : " + TotalUnits());

            decimal grand = 0m;
            for (int w = 0; w < WarehouseCount; w++) grand += WarehouseValue(w);
            Console.WriteLine("Total stock value  : " + grand.ToString("N2"));
            Console.WriteLine("Movements recorded : " + logCount);
        }

        // =====================================================
        // CALCULATION HELPERS
        // =====================================================
        static int ProductUnits(int p)
        {
            int sum = 0;
            for (int w = 0; w < WarehouseCount; w++) sum += qty[p, w];
            return sum;
        }

        static int WarehouseUnits(int w)
        {
            int sum = 0;
            for (int p = 0; p < productCount; p++) sum += qty[p, w];
            return sum;
        }

        static decimal WarehouseValue(int w)
        {
            decimal sum = 0m;
            for (int p = 0; p < productCount; p++) sum += qty[p, w] * prices[p];
            return sum;
        }

        static int TotalUnits()
        {
            int sum = 0;
            for (int w = 0; w < WarehouseCount; w++) sum += WarehouseUnits(w);
            return sum;
        }

        static int FindByCode(string code)
        {
            for (int i = 0; i < productCount; i++)
                if (string.Equals(codes[i], code, StringComparison.OrdinalIgnoreCase)) return i;
            return -1;
        }

        static void AddLog(string type, string code, int amount, string detail)
        {
            // If the log is full, drop the oldest record
            if (logCount == MaxLog)
            {
                for (int i = 1; i < MaxLog; i++)
                {
                    logTime[i - 1] = logTime[i];
                    logType[i - 1] = logType[i];
                    logCode[i - 1] = logCode[i];
                    logQty[i - 1] = logQty[i];
                    logDetail[i - 1] = logDetail[i];
                }
                logCount--;
            }

            logTime[logCount] = DateTime.Now.ToString("yyyy-MM-dd HH:mm");
            logType[logCount] = type;
            logCode[logCount] = code;
            logQty[logCount] = amount;
            logDetail[logCount] = detail;
            logCount++;
        }

        // =====================================================
        // DISPLAY HELPERS
        // =====================================================
        static void Header(string title)
        {
            Console.WriteLine("--- " + title + " ---");
            Console.WriteLine();
        }

        static void PrintProductHeader()
        {
            string line = "Code".PadRight(8) + "Name".PadRight(22) + "Price".PadLeft(10);
            for (int w = 0; w < WarehouseCount; w++) line += WarehouseNames[w].Replace("Warehouse ", "WH-").PadLeft(7);
            line += "Total".PadLeft(8);
            Console.WriteLine(line);
            Console.WriteLine(new string('-', line.Length));
        }

        static void PrintProductRow(int p)
        {
            string line = codes[p].PadRight(8) + names[p].PadRight(22) + prices[p].ToString("N2").PadLeft(10);
            for (int w = 0; w < WarehouseCount; w++) line += qty[p, w].ToString().PadLeft(7);
            line += ProductUnits(p).ToString().PadLeft(8);
            Console.WriteLine(line);
        }

        // =====================================================
        // INPUT HELPERS (all validate and re-prompt)
        // =====================================================
        static string ReadText(string prompt)
        {
            while (true)
            {
                Console.Write(prompt);
                string s = (Console.ReadLine() ?? "").Trim();
                if (s.Length > 0) return s;
                Console.WriteLine("Input cannot be empty.");
            }
        }

        static int ReadInt(string prompt, int min, int max)
        {
            while (true)
            {
                Console.Write(prompt);
                string s = Console.ReadLine();
                if (int.TryParse(s, out int value) && value >= min && value <= max) return value;
                Console.WriteLine("Please enter a whole number" +
                                  (max == int.MaxValue ? " of at least " + min : " from " + min + " to " + max) + ".");
            }
        }

        static decimal ReadDecimal(string prompt, decimal min)
        {
            while (true)
            {
                Console.Write(prompt);
                string s = Console.ReadLine();
                if (decimal.TryParse(s, NumberStyles.Number, CultureInfo.InvariantCulture, out decimal value) && value >= min)
                    return value;
                Console.WriteLine("Please enter a valid amount of at least " + min + ".");
            }
        }

        static int ChooseWarehouse(string label)
        {
            Console.WriteLine(label + ":");
            for (int w = 0; w < WarehouseCount; w++)
                Console.WriteLine("  " + (w + 1) + ". " + WarehouseNames[w]);
            return ReadInt("Warehouse number: ", 1, WarehouseCount) - 1;
        }

        static int AskForExistingProduct()
        {
            string code = ReadText("Product code: ");
            int p = FindByCode(code);
            if (p < 0)
            {
                Console.WriteLine("Product '" + code + "' not found.");
                return -1;
            }
            Console.WriteLine("Product: " + names[p]);
            return p;
        }

        // =====================================================
        // SAMPLE DATA (so the menu has something to show)
        // =====================================================
        static void SeedSampleData()
        {
            AddSeed("P1", "Monitor", 1250m, 40, 25, 8);
            AddSeed("P2", "Keyboard", 95m, 120, 60, 30);
            AddSeed("P3", "Laptop", 28m, 200, 5, 90);
            AddSeed("P4", "Mouse", 12m, 6, 15, 0);
        }

        static void AddSeed(string code, string name, decimal price, int a, int b, int c)
        {
            int p = productCount++;
            codes[p] = code;
            names[p] = name;
            prices[p] = price;
            qty[p, 0] = a;
            qty[p, 1] = b;
            qty[p, 2] = c;
            AddLog("RECEIVE", code, a + b + c, "Initial stock (A=" + a + ", B=" + b + ", C=" + c + ")");
        }
    }
}
