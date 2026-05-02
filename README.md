// =====================================================
//  TELEGRAM SUPPORT BOT (SMILE System - Complete)
//  Working: Points | Stamps | Cashier | AddPoints
// =====================================================

// ----- 1. CONFIGURATION -----
var BOT_TOKEN = '8360800911:AAHGU_IdthYAoseZnNlP0nXloG0ly74bXSk';
var TELEGRAM_API_URL = 'https://api.telegram.org/bot' + BOT_TOKEN;
var ADMIN_CHAT_ID = '5114376674';
var MAX_SEARCH_RESULTS = 5;
var SEARCH_TIMEOUT_MS = 5000;

// ----- 2. HELPER FUNCTIONS -----
function getSheetData(sheet, startRow) {
  if (!sheet) return [];
  var lastRow = sheet.getLastRow();
  if (lastRow < startRow) return [];
  var lastCol = sheet.getLastColumn();
  if (lastCol < 1) return [];
  return sheet.getRange(startRow, 1, lastRow - startRow + 1, lastCol).getValues();
}

// ----- 3. GET USER ROLE -----
function getUserRole(chatId) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("TELEGRAM_USERS");
  if (!sheet) return "SMILER";
  
  var data = getSheetData(sheet, 2);
  for (var i = 0; i < data.length; i++) {
    var tgId = String(data[i][3]).trim();
    if (tgId === String(chatId)) {
      var role = String(data[i][5]).trim().toUpperCase();
      if (role === "SUPER" || role === "AUDITOR") return "SUPER";
      if (role === "LEADER") return "LEADER";
      if (role === "RANGER") return "RANGER";
      return "SMILER";
    }
  }
  return "SMILER";
}

// ----- 4. GET IMAGE URL FROM PRODUCT_CATALOG (Column C) -----
function getProductImageUrl(skid) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("PRODUCT_CATALOG");
  if (!sheet) return null;
  
  var data = getSheetData(sheet, 2);
  var upperSkid = skid.toUpperCase().trim();
  var IMAGE_COLUMN_INDEX = 2;
  
  for (var i = 0; i < data.length; i++) {
    var catalogId = String(data[i][0]).trim().toUpperCase();
    if (catalogId === upperSkid || catalogId.indexOf(upperSkid + "_") === 0) {
      var imageUrl = data[i][IMAGE_COLUMN_INDEX];
      if (imageUrl && imageUrl !== "" && imageUrl !== "-" && imageUrl !== "#N/A") {
        return imageUrl;
      }
    }
  }
  return null;
}

// ----- 5. GET CATALOG DATA FROM PRODUCT_CATALOG -----
function getCatalogData() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("PRODUCT_CATALOG");
  var catalogMap = {};
  
  if (!sheet) return catalogMap;
  
  var data = getSheetData(sheet, 2);
  for (var i = 0; i < data.length; i++) {
    var catalogId = String(data[i][0]).trim();
    var displayName = data[i][1];
    var category = data[i][3];
    
    if (catalogId) {
      catalogMap[catalogId] = {
        displayName: displayName,
        category: category
      };
    }
  }
  return catalogMap;
}

// =====================================================
// ========== FUZZY SEARCH ==========
// =====================================================

function normalizeText(text) {
  return text.toLowerCase().replace(/\s+/g, ' ').trim();
}

function tokenizeImproved(text) {
  var normalized = normalizeText(text);
  var words = normalized.split(/\s+/);
  var tokens = [];
  
  for (var i = 0; i < words.length; i++) {
    if (words[i].length >= 2) {
      tokens.push(words[i]);
    }
  }
  
  var noSpace = normalized.replace(/\s/g, '');
  if (noSpace.length >= 3 && noSpace !== normalized) {
    tokens.push(noSpace);
  }
  return tokens;
}

function calculateMatchScoreImproved(productName, searchTokens) {
  var nameLower = productName.toLowerCase();
  var nameNoSpace = nameLower.replace(/\s/g, '');
  var score = 0;
  
  for (var i = 0; i < searchTokens.length; i++) {
    var token = searchTokens[i];
    if (token.length < 2) continue;
    
    if (nameLower.indexOf(token) !== -1) {
      score += 10;
    }
    if (nameNoSpace.indexOf(token) !== -1 && token.length >= 3) {
      score += 8;
    }
    var wordBoundary = new RegExp('\\b' + token.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + '\\b', 'i');
    if (wordBoundary.test(nameLower)) {
      score += 15;
    }
  }
  return score;
}

function fuzzySearchProductsImproved(searchText) {
  var startTime = new Date().getTime();
  var searchTokens = tokenizeImproved(searchText);
  if (searchTokens.length === 0) return [];
  
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("MASTER_INVENTORY");
  if (!sheet) return [];
  
  var data = getSheetData(sheet, 2);
  var results = [];
  
  for (var i = 0; i < data.length; i++) {
    if (new Date().getTime() - startTime > SEARCH_TIMEOUT_MS) break;
    
    var nameInSheet = String(data[i][2]);
    var score = calculateMatchScoreImproved(nameInSheet, searchTokens);
    
    if (score > 0) {
      results.push({
        skid: data[i][0],
        name: data[i][2],
        category: data[i][4],
        retailPrice: data[i][7],
        wholesalePrice: data[i][6],
        purchasePrice: data[i][5],
        ypStock: Number(data[i][8]) || 0,
        gmpStock: Number(data[i][9]) || 0,
        totalStock: Number(data[i][10]) || 0,
        score: score
      });
    }
  }
  
  var grouped = {};
  for (var i = 0; i < results.length; i++) {
    var r = results[i];
    var skid = String(r.skid).toUpperCase().trim();
    if (!grouped[skid]) {
      grouped[skid] = {
        skid: r.skid,
        name: r.name,
        category: r.category,
        retailPrice: r.retailPrice,
        wholesalePrice: r.wholesalePrice,
        purchasePrice: r.purchasePrice,
        ypStock: 0,
        gmpStock: 0,
        totalStock: 0,
        score: r.score
      };
    }
    grouped[skid].ypStock += r.ypStock;
    grouped[skid].gmpStock += r.gmpStock;
    grouped[skid].totalStock += r.totalStock;
    grouped[skid].score = Math.max(grouped[skid].score, r.score);
  }
  
  var groupedResults = [];
  for (var skid in grouped) {
    groupedResults.push(grouped[skid]);
  }
  groupedResults.sort(function(a, b) { return b.score - a.score; });
  return groupedResults.slice(0, MAX_SEARCH_RESULTS * 2);
}

// =====================================================
// ========== PRODUCT GROUPING ==========
// =====================================================

function getProductGroup(skid) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("MASTER_INVENTORY");
  if (!sheet) return null;
  
  var data = getSheetData(sheet, 2);
  var searchSkid = skid.toUpperCase().trim();
  var products = [];
  
  for (var i = 0; i < data.length; i++) {
    var rowSkid = String(data[i][0]).trim().toUpperCase();
    if (rowSkid === searchSkid) {
      var colour = String(data[i][3]).trim();
      if (colour !== "Standard") {
        products.push({
          skid: data[i][0],
          name: data[i][2],
          colour: colour,
          category: data[i][4],
          purchasePrice: data[i][5],
          wholesalePrice: data[i][6],
          retailPrice: data[i][7],
          ypStock: Number(data[i][8]) || 0,
          gmpStock: Number(data[i][9]) || 0,
          totalStock: Number(data[i][10]) || 0,
          status: data[i][11]
        });
      }
    }
  }
  
  if (products.length === 0) return null;
  
  var ypColors = {};
  var gmpColors = {};
  var allColors = [];
  var totalStock = 0;
  var ypStockTotal = 0;
  var gmpStockTotal = 0;
  
  var name = products[0].name;
  var category = products[0].category;
  var retailPrice = products[0].retailPrice;
  var wholesalePrice = products[0].wholesalePrice;
  var purchasePrice = products[0].purchasePrice;
  
  for (var i = 0; i < products.length; i++) {
    var p = products[i];
    var colour = p.colour;
    
    ypStockTotal += p.ypStock;
    gmpStockTotal += p.gmpStock;
    totalStock += p.totalStock;
    
    if (allColors.indexOf(colour) === -1) allColors.push(colour);
    
    if (p.ypStock > 0) {
      ypColors[colour] = (ypColors[colour] || 0) + p.ypStock;
    }
    if (p.gmpStock > 0) {
      gmpColors[colour] = (gmpColors[colour] || 0) + p.gmpStock;
    }
  }
  
  return {
    skid: searchSkid,
    name: name,
    category: category,
    retailPrice: retailPrice,
    wholesalePrice: wholesalePrice,
    purchasePrice: purchasePrice,
    ypStock: ypStockTotal,
    gmpStock: gmpStockTotal,
    totalStock: totalStock,
    ypColors: ypColors,
    gmpColors: gmpColors,
    allColors: allColors,
    hasYP: ypStockTotal > 0,
    hasGMP: gmpStockTotal > 0
  };
}

// =====================================================
// ========== SEND MESSAGE FUNCTIONS ==========
// =====================================================

function sendMessage(chatId, text) {
  if (!text) return;
  var url = TELEGRAM_API_URL + "/sendMessage";
  var payload = {
    'chat_id': String(chatId),
    'text': text,
    'parse_mode': 'Markdown'
  };
  var options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(payload)
  };
  UrlFetchApp.fetch(url, options);
}

function sendChatAction(chatId, action) {
  var url = TELEGRAM_API_URL + "/sendChatAction";
  var payload = { 'chat_id': String(chatId), 'action': action };
  var options = { 'method': 'post', 'payload': payload };
  UrlFetchApp.fetch(url, options);
}

function sendMessageWithKeyboard(chatId, text, keyboard) {
  var url = TELEGRAM_API_URL + "/sendMessage";
  var payload = {
    'chat_id': String(chatId),
    'text': text,
    'parse_mode': 'Markdown',
    'reply_markup': JSON.stringify(keyboard)
  };
  var options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(payload)
  };
  UrlFetchApp.fetch(url, options);
}

function sendMessageWithProductButtons(chatId, text, products) {
  var url = TELEGRAM_API_URL + "/sendMessage";
  
  var buttons = [];
  for (var i = 0; i < Math.min(products.length, 5); i++) {
    var p = products[i];
    var displayName = p.name;
    if (displayName.length > 30) {
      displayName = displayName.substring(0, 27) + "...";
    }
    buttons.push([{
      "text": "📱 " + displayName,
      "callback_data": "PROD_" + p.skid
    }]);
  }
  
  buttons.push([{"text": "🔄 အခြားနည်းဖြင့် ရှာရန်", "callback_data": "HELP_SEARCH"}]);
  buttons.push([{"text": "🏠 ပင်မစာမျက်နှာ", "callback_data": "START"}]);
  
  var keyboard = { "inline_keyboard": buttons };
  
  var payload = {
    'chat_id': String(chatId),
    'text': text,
    'parse_mode': 'Markdown',
    'reply_markup': JSON.stringify(keyboard)
  };
  
  var options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(payload)
  };
  UrlFetchApp.fetch(url, options);
}

function sendSingleProductWithButton(chatId, text, group) {
  var url = TELEGRAM_API_URL + "/sendMessage";
  
  var displayName = group.name;
  if (displayName.length > 35) {
    displayName = displayName.substring(0, 32) + "...";
  }
  
  var keyboard = {
    "inline_keyboard": [
      [
        {"text": "🔍 " + displayName + " (ပြန်ရှာရန်)", "callback_data": "PROD_" + group.skid}
      ],
      [
        {"text": "📞 ဆက်သွယ်ရန်", "callback_data": "CONTACT_MENU"}
      ],
      [
        {"text": "🏠 ပင်မစာမျက်နှာ", "callback_data": "START"}
      ]
    ]
  };
  
  var payload = {
    'chat_id': String(chatId),
    'text': text,
    'parse_mode': 'Markdown',
    'reply_markup': JSON.stringify(keyboard)
  };
  
  var options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(payload)
  };
  UrlFetchApp.fetch(url, options);
}

function sendProductImageWithCaption(chatId, imageUrl, caption, group) {
  var url = TELEGRAM_API_URL + "/sendPhoto";
  
  var payload = {
    'chat_id': String(chatId),
    'photo': imageUrl,
    'caption': caption,
    'parse_mode': 'Markdown'
  };
  
  try {
    var options = { 'method': 'post', 'payload': payload };
    UrlFetchApp.fetch(url, options);
  } catch(e) {
    Logger.log("Send photo error: " + e.toString());
    sendMessage(chatId, caption);
  }
  
  sendContactButtons(chatId, group);
}

function sendContactButtons(chatId, group) {
  var displayName = group.name;
  if (displayName.length > 30) {
    displayName = displayName.substring(0, 27) + "...";
  }
  
  var keyboard = {
    "inline_keyboard": [
      [
        {"text": "🔍 " + displayName + " (ပြန်ရှာရန်)", "callback_data": "PROD_" + group.skid}
      ],
      [
        {"text": "📞 ဆက်သွယ်ရန်", "callback_data": "CONTACT_MENU"}
      ],
      [
        {"text": "🏠 ပင်မစာမျက်နှာ", "callback_data": "START"}
      ]
    ]
  };
  
  sendMessageWithKeyboard(chatId, "📌 *ဆက်လက်လုပ်ဆောင်ရန်*", keyboard);
}

// =====================================================
// ========== CONTACT HANDLERS ==========
// =====================================================

function sendContactMenu(chatId) {
  var keyboard = {
    "inline_keyboard": [
      [
        {"text": "📞 Phone ခေါ်ဆိုရန်", "callback_data": "CONTACT_PHONE"},
        {"text": "💬 Viber ဖွင့်ရန်", "callback_data": "CONTACT_VIBER"}
      ],
      [
        {"text": "📱 Telegram ဖွင့်ရန်", "callback_data": "CONTACT_TELEGRAM"},
        {"text": "◀️ နောက်သို့", "callback_data": "START"}
      ]
    ]
  };
  
  sendMessageWithKeyboard(chatId, "📞 *ဆက်သွယ်ရန် နည်းလမ်းရွေးပါ*", keyboard);
}

function sendPhoneContact(chatId) {
  var message = "📞 *ဖုန်းဖြင့် ဆက်သွယ်ရန်*\n\n" +
                "• ယုဇနပလာဇာ: [09 780001662](tel:+959780001662)\n" +
                "• ဂမုန်းပွင့် (ကန်တော်လေး): [09 780001682](tel:+959780001682)\n\n" +
                "📱 နံပါတ်ကို နှိပ်လိုက်ရုံဖြင့် ဖုန်းခေါ်ဆိုနိုင်ပါသည်။";
  sendMessage(chatId, message);
}

function sendViberContact(chatId) {
  var message = "💬 *Viber ဖြင့် ဆက်သွယ်ရန်*\n\n" +
                "• ယုဇနပလာဇာ: [Viber ဖွင့်ရန်](viber://chat?number=+959780001662)\n" +
                "• ဂမုန်းပွင့် (ကန်တော်လေး): [Viber ဖွင့်ရန်](viber://chat?number=+959780001682)\n\n" +
                "📱 နှိပ်လိုက်ရုံဖြင့် Viber ဖွင့်ပေးပါမည်။";
  sendMessage(chatId, message);
}

function sendTelegramContact(chatId) {
  var message = "📱 *Telegram ဖြင့် ဆက်သွယ်ရန်*\n\n" +
                "• ယုဇနပလာဇာ: [Telegram ဖွင့်ရန်](https://t.me/yuzana_plaza)\n" +
                "• ဂမုန်းပွင့် (ကန်တော်လေး): [Telegram ဖွင့်ရန်](https://t.me/kantaw_ley)\n\n" +
                "📱 နှိပ်လိုက်ရုံဖြင့် Telegram ဖွင့်ပေးပါမည်။";
  sendMessage(chatId, message);
}

// =====================================================
// ========== POINTS & STAMPS SYSTEM ==========
// =====================================================

function getPointsBalance(chatId) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  if (!sheet) return 0;
  
  var data = getSheetData(sheet, 2);
  var totalPointsIn = 0;
  var totalPointsOut = 0;
  
  for (var i = 0; i < data.length; i++) {
    if (String(data[i][2]) === String(chatId)) {
      totalPointsIn += Number(data[i][4]) || 0;
      totalPointsOut += Number(data[i][5]) || 0;
    }
  }
  return totalPointsIn - totalPointsOut;
}

function getPointsHistory(chatId, limit) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  if (!sheet) return [];
  
  var data = getSheetData(sheet, 2);
  var history = [];
  
  for (var i = data.length - 1; i >= 0 && history.length < (limit || 10); i--) {
    if (String(data[i][2]) === String(chatId)) {
      history.push({
        date: data[i][0],
        pointsIn: data[i][4] || 0,
        pointsOut: data[i][5] || 0,
        category: data[i][6],
        description: data[i][8]
      });
    }
  }
  return history;
}

function checkPointsCustomerExists(chatId) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  if (!sheet) return false;
  
  var data = getSheetData(sheet, 2);
  for (var i = 0; i < data.length; i++) {
    if (String(data[i][2]) === String(chatId)) {
      return true;
    }
  }
  return false;
}

function registerPointsCustomer(chatId, phoneNumber, customerName) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  
  if (!sheet) {
    sheet = ss.insertSheet("POINT_LEDGER");
    sheet.appendRow(["Timestamp", "Phone Number", "Customer ID", "Customer Name", 
                     "Points In", "Points Out", "Category", "Ref_Invoice", 
                     "Description", "Cashier Email", "Branch", "Sync_Status"]);
  }
  
  if (checkPointsCustomerExists(chatId)) {
    return { success: false, message: "✅ သင်သည် စာရင်းသွင်းပြီးသားဖြစ်ပါသည်။\n📊 `/points` ဖြင့် စစ်ဆေးပါ။" };
  }
  
  sheet.appendRow([
    new Date(),
    phoneNumber,
    chatId,
    customerName,
    0,
    0,
    "REGISTER",
    "REG_" + chatId,
    "New registration",
    "telegram_bot",
    "TG_BOT",
    "SYNCED"
  ]);
  
  return { success: true, message: "🎉 *စာရင်းသွင်းခြင်း အောင်မြင်ပါသည်!*\n\n📞 ဖုန်း: " + phoneNumber + "\n💎 *Starting Points:* 0\n\n🛍️ ဝယ်ယူမှုတိုင်းအတွက် Points များ စုဆောင်းနိုင်ပါသည်။\n📌 `/points` - သင့် Points များကို ကြည့်ရှုပါ။" };
}

function formatPointsMessage(balance, history) {
  var message = "💎 *POINTS LEDGER*\n";
  message += "━━━━━━━━━━━━━━━━━━━━━\n";
  message += "📊 *လက်ကျန် Points:* " + balance + "\n";
  message += "━━━━━━━━━━━━━━━━━━━━━\n\n";
  
  if (history && history.length > 0) {
    message += "📜 *နောက်ဆုံး လှုပ်ရှားမှုများ:*\n";
    for (var i = 0; i < Math.min(history.length, 5); i++) {
      var h = history[i];
      var date = new Date(h.date).toLocaleDateString();
      if (h.pointsIn > 0) {
        message += "✅ " + date + ": +" + h.pointsIn + " points (" + h.category + ")\n";
      } else if (h.pointsOut > 0) {
        message += "🔄 " + date + ": -" + h.pointsOut + " points (" + h.category + ")\n";
      }
    }
    message += "\n";
  }
  
  message += "💡 `/stamp` - Stamps စစ်ရန်\n";
  message += "💡 `/status` - Card အဆင့်စစ်ရန်";
  return message;
}

// Get Customer Stamp Info
function getCustomerStampInfo(chatId) {
  var spending = getCustomerTotalSpending(chatId);
  var stamps = Math.floor(spending.total / 30000);
  return { stampCount: Math.min(stamps, 30), totalSpending: spending.total };
}

function getCustomerTotalSpending(chatId) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  if (!sheet) return { total: 0, lastPurchase: 0 };
  
  var data = getSheetData(sheet, 2);
  var totalSpending = 0;
  var lastPurchaseAmount = 0;
  
  for (var i = 0; i < data.length; i++) {
    if (String(data[i][2]) === String(chatId)) {
      var desc = String(data[i][8]);
      var amountMatch = desc.match(/(\d+[,]?\d*)/);
      if (amountMatch) {
        var amount = parseFloat(amountMatch[0].replace(/,/g, ''));
        if (!isNaN(amount)) {
          totalSpending += amount;
          lastPurchaseAmount = amount;
        }
      }
    }
  }
  return { total: totalSpending, lastPurchase: lastPurchaseAmount };
}

function getCurrentCardTier(totalSpending) {
  if (totalSpending >= 900000) return { tier: "VVIP", code: "VVIP", color: "💎", nextNeeded: 0, discount: 5 };
  if (totalSpending >= 630000) return { tier: "Diamond", code: "DIAMOND", color: "🥇", nextNeeded: 900000 - totalSpending, discount: 0 };
  if (totalSpending >= 330000) return { tier: "Platinum", code: "PLATINUM", color: "🥈", nextNeeded: 630000 - totalSpending, discount: 0 };
  if (totalSpending >= 30000) return { tier: "Gold", code: "GOLD", color: "🥉", nextNeeded: 330000 - totalSpending, discount: 0 };
  return { tier: "Standard", code: "STANDARD", color: "⚪", nextNeeded: 30000, discount: 0 };
}

function formatStampStatus(chatId) {
  var stampInfo = getCustomerStampInfo(chatId);
  var tier = getCurrentCardTier(stampInfo.totalSpending);
  
  var message = "📮 *STAMP MEMBERSHIP*\n";
  message += "━━━━━━━━━━━━━━━━━━━━━━━\n\n";
  message += tier.color + " *" + tier.tier + " Member*\n";
  message += "━━━━━━━━━━━━━━━━━━━━━━━\n";
  message += "📮 Stamps: " + stampInfo.stampCount + "/30\n";
  
  var barLength = 20;
  var filledBars = Math.floor((stampInfo.stampCount / 30) * barLength);
  var emptyBars = barLength - filledBars;
  message += "```";
  for (var i = 0; i < filledBars; i++) message += "📮";
  for (var i = 0; i < emptyBars; i++) message += "◻️";
  message += "```\n\n";
  
  message += "💰 စုစုပေါင်းအသုံးစရိတ်: " + formatPrice(stampInfo.totalSpending) + " Ks\n";
  
  if (stampInfo.stampCount >= 30) {
    message += "\n🎉 *VVIP Achieved!* 🎉\n💳 နောက်ဆက်တွဲဝယ်ယူမှုတိုင်း 5% Discount\n";
  } else {
    var needed = 30 - stampInfo.stampCount;
    message += "\n✨ VVIP သို့တက်ရန်: " + needed + " stamps လိုသည်\n";
    message += "   (ဝယ်ယူမှု " + formatPrice(needed * 30000) + " Ks)\n";
  }
  
  message += "\n━━━━━━━━━━━━━━━━━━━━━━━\n";
  message += "💡 30,000 Ks = 1 Stamp\n";
  message += "💡 1 year validity\n";
  message += "💡 `/points` - Points စစ်ရန်";
  return message;
}

function formatCardStatus(chatId) {
  var spending = getCustomerTotalSpending(chatId);
  var currentTier = getCurrentCardTier(spending.total);
  
  var message = "💳 *MY CARD STATUS*\n";
  message += "━━━━━━━━━━━━━━━━━━━━━━━\n\n";
  message += currentTier.color + " *" + currentTier.tier + " Card*\n";
  message += "━━━━━━━━━━━━━━━━━━━━━━━\n";
  message += "💰 စုစုပေါင်းအသုံးစရိတ်: " + formatPrice(spending.total) + " Ks\n\n";
  
  if (currentTier.nextNeeded > 0 && currentTier.tier !== "VVIP") {
    var nextTierName = "";
    var nextTierColor = "";
    if (currentTier.tier === "Gold") { nextTierName = "Platinum"; nextTierColor = "🥈"; }
    else if (currentTier.tier === "Platinum") { nextTierName = "Diamond"; nextTierColor = "🥇"; }
    else if (currentTier.tier === "Diamond") { nextTierName = "VVIP"; nextTierColor = "💎"; }
    
    message += "✨ *နောက်အဆင့်သို့တက်ရန်*:\n";
    message += "   " + nextTierColor + " " + nextTierName + " Card\n";
    message += "   လိုအပ်သောငွေ: " + formatPrice(currentTier.nextNeeded) + " Ks\n\n";
  }
  
  if (currentTier.tier === "VVIP") {
    message += "🎁 *အကျိုးခံစားခွင့်:* 5% Discount\n";
  }
  
  message += "━━━━━━━━━━━━━━━━━━━━━━━\n";
  message += "💡 `/stamp` - Stamps စစ်ရန်\n";
  message += "💡 `/points` - Points စစ်ရန်";
  return message;
}

// =====================================================
// ========== CASHIER FUNCTIONS ==========
// =====================================================

function getChatIdByPhone(phone) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  if (!sheet) return null;
  
  var data = getSheetData(sheet, 2);
  for (var i = data.length - 1; i >= 0; i--) {
    if (String(data[i][1]) === String(phone) && data[i][2]) {
      return String(data[i][2]);
    }
  }
  return null;
}

function addPointsToLedger(phone, chatId, points, amount, invoice) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("POINT_LEDGER");
  
  if (!sheet) {
    sheet = ss.insertSheet("POINT_LEDGER");
    sheet.appendRow(["Timestamp", "Phone Number", "Customer ID", "Customer Name", 
                     "Points In", "Points Out", "Category", "Ref_Invoice", 
                     "Description", "Cashier Email", "Branch", "Sync_Status"]);
  }
  
  sheet.appendRow([
    new Date(),
    phone,
    chatId,
    "",
    points,
    0,
    "PURCHASE",
    invoice,
    amount + " MMK",
    "telegram_cashier",
    "TG_BOT",
    "SYNCED"
  ]);
}

// =====================================================
// ========== USER STATE MANAGEMENT ==========
// =====================================================

function setUserState(chatId, state) {
  var scriptProperties = PropertiesService.getScriptProperties();
  scriptProperties.setProperty('user_state_' + chatId, state);
}

function getUserState(chatId) {
  var scriptProperties = PropertiesService.getScriptProperties();
  return scriptProperties.getProperty('user_state_' + chatId);
}

function clearUserState(chatId) {
  var scriptProperties = PropertiesService.getScriptProperties();
  scriptProperties.deleteProperty('user_state_' + chatId);
}

// =====================================================
// ========== AI ASSISTANT ==========
// =====================================================

function getGeminiApiKey() {
  var scriptProperties = PropertiesService.getScriptProperties();
  return scriptProperties.getProperty('GEMINI_API_KEY');
}

function isAIConfigured() {
  return getGeminiApiKey() !== null;
}

function askAIAboutInventory(question) {
  var GEMINI_API_KEY = getGeminiApiKey();
  if (!GEMINI_API_KEY) {
    return "🤖 AI Assistant ကို စီစဉ်သတ်မှတ်ထားခြင်း မရှိသေးပါ။\n\n📌 *ပြင်ဆင်နည်း:* Script Properties တွင် GEMINI_API_KEY ထည့်ပါ။";
  }
  
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sheet = ss.getSheetByName("MASTER_INVENTORY");
    if (!sheet) return "Inventory data not found.";
    
    var data = sheet.getDataRange().getValues();
    var context = "INVENTORY DATA:\n";
    var productCount = 0;
    
    for (var i = 1; i < data.length && productCount < 30; i++) {
      var row = data[i];
      var stock = Number(row[10]) || 0;
      if (stock > 0) {
        context += "- " + row[2] + " (SKID: " + row[0] + ") - " + formatPrice(row[7]) + " Ks, Stock: " + stock + "\n";
        productCount++;
      }
    }
    
    var prompt = "You are a helpful shop assistant. Answer the user's question based ONLY on the inventory data below.\n\n" +
                 context + "\n\n" +
                 "USER QUESTION: " + question + "\n\n" +
                 "INSTRUCTIONS:\n" +
                 "1. Answer only based on the data above\n" +
                 "2. If data doesn't contain the answer, say 'စာရင်းထဲတွင် မပါရှိပါ'\n" +
                 "3. Be concise and helpful\n" +
                 "4. Use Burmese language (မြန်မာလို)\n" +
                 "5. Format prices with commas (e.g., 740,000 Ks)";
    
    var url = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=' + GEMINI_API_KEY;
    var payload = { "contents": [{ "parts": [{ "text": prompt }] }] };
    var options = { 'method': 'post', 'contentType': 'application/json', 'payload': JSON.stringify(payload), 'muteHttpExceptions': true };
    var response = UrlFetchApp.fetch(url, options);
    var responseData = JSON.parse(response.getContentText());
    
    if (responseData.candidates && responseData.candidates[0] && responseData.candidates[0].content) {
      return responseData.candidates[0].content.parts[0].text;
    }
    return "🤖 ကျေးဇူးပြု၍ နောက်မှ ထပ်မံကြိုးစားပါ။";
  } catch (error) {
    Logger.log("AI Error: " + error.toString());
    return "🤖 AI ဝန်ဆောင်မှုတွင် ယာယီအမှားတစ်ခုဖြစ်ပွားနေပါသည်။\n\nကျေးဇူးပြု၍ နောက်မှထပ်မံကြိုးစားပါ။";
  }
}

function shouldUseAI(text) {
  var aiKeywords = ["ဘယ်လိုလဲ", "ဘယ်နှစ်လုံး", "အကြံပေး", "ဘယ်ဟာကောင်းလဲ", "ဘာကွာလဲ", "အကြောင်း", "အကောင်းဆုံး"];
  var lowerText = text.toLowerCase();
  for (var i = 0; i < aiKeywords.length; i++) {
    if (lowerText.indexOf(aiKeywords[i]) !== -1) {
      return true;
    }
  }
  return false;
}

// =====================================================
// ========== SMART REPLY ==========
// =====================================================

function getSmartReply(text, role) {
  var searchText = text.trim();
  if (searchText.startsWith('/')) return null;
  
  var exactGroup = getProductGroup(searchText.toUpperCase());
  if (exactGroup && exactGroup.totalStock > 0) {
    return { type: 'single', group: exactGroup };
  }
  
  var fuzzyResults = fuzzySearchProductsImproved(searchText);
  var validResults = [];
  for (var i = 0; i < fuzzyResults.length; i++) {
    if (fuzzyResults[i].totalStock > 0) {
      validResults.push(fuzzyResults[i]);
    }
  }
  
  if (validResults.length === 0) {
    return { type: 'none', message: "❌ `" + searchText + "` နှင့် ဆက်စပ်သော လက်ကျန်ရှိပစ္စည်း မတွေ့ပါ။\n\n💡 ပိုမိုတိကျစွာရိုက်ထည့်ပါက ရှာဖွေမှု အသေးစိတ်ကူညီနိုင်ပါသည်။" };
  }
  
  if (validResults.length === 1) {
    var singleGroup = getProductGroup(validResults[0].skid);
    if (singleGroup && singleGroup.totalStock > 0) {
      return { type: 'single', group: singleGroup };
    }
  }
  
  return { type: 'multiple', products: validResults, searchText: searchText, role: role };
}

// =====================================================
// ========== FORMAT MESSAGES ==========
// =====================================================

function formatPrice(price) {
  if (!price || price === 0) return "0";
  return price.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
}

function formatSingleProduct(group, role) {
  if (!group) return null;
  if (group.totalStock === 0) return null;
  
  var message = "";
  message += "🔍 *" + group.skid + "*\n";
  message += "━━━━━━━━━━━━━━━━━━━━━\n";
  message += "📱 " + group.name + "\n";
  
  if (role === "SMILER") {
    message += "💰 " + formatPrice(group.retailPrice) + " Ks\n";
    
    if (group.hasYP || group.hasGMP) {
      message += "📦 *ရရှိနိုင်သောဆိုင်:*\n";
      
      if (group.hasYP) {
        var ypColorsList = [];
        for (var color in group.ypColors) {
          ypColorsList.push(color);
        }
        message += "🏪 *ယုဇနပလာဇာ*: " + ypColorsList.join(", ") + "\n";
      }
      
      if (group.hasGMP) {
        var gmpColorsList = [];
        for (var color in group.gmpColors) {
          gmpColorsList.push(color);
        }
        message += "🏪 *ဂမုန်းပွင့် (ကန်တော်လေး)*: " + gmpColorsList.join(", ") + "\n";
      }
    } else {
      message += "📦 အခြေအနေ: လက်ကျန်မရှိပါ\n";
    }
  }
  
  else if (role === "RANGER") {
    message += "💰 " + formatPrice(group.retailPrice) + " Ks\n";
    
    if (group.hasYP || group.hasGMP) {
      message += "📦 *ရရှိနိုင်သောဆိုင်:*\n";
      
      if (group.hasYP) {
        var ypColorsList = [];
        for (var color in group.ypColors) {
          ypColorsList.push(color);
        }
        message += "🏪 *ယုဇနပလာဇာ*: " + ypColorsList.join(", ") + "\n";
      }
      
      if (group.hasGMP) {
        var gmpColorsList = [];
        for (var color in group.gmpColors) {
          gmpColorsList.push(color);
        }
        message += "🏪 *ဂမုန်းပွင့် (ကန်တော်လေး)*: " + gmpColorsList.join(", ") + "\n";
      }
    } else {
      message += "📦 အခြေအနေ: လက်ကျန်မရှိပါ\n";
    }
  }
  
  else if (role === "LEADER") {
    message += "💰 *ဈေးနှုန်းများ*\n";
    message += "   🏷️ လက်ကား : " + formatPrice(group.wholesalePrice) + " Ks\n";
    message += "   🏷️ လက်လီ : " + formatPrice(group.retailPrice) + " Ks\n";
    message += "\n📦 *လက်ကျန်အသေးစိတ်*\n";
    
    if (group.hasYP) {
      message += "🏪 *ယုဇနပလာဇာ*\n";
      for (var color in group.ypColors) {
        message += "   🎨 " + color + " : " + group.ypColors[color] + " ခု\n";
      }
    }
    if (group.hasGMP) {
      message += "🏪 *ဂမုန်းပွင့် (ကန်တော်လေး)*\n";
      for (var color in group.gmpColors) {
        message += "   🎨 " + color + " : " + group.gmpColors[color] + " ခု\n";
      }
    }
    message += "📊 စုစုပေါင်း : " + group.totalStock + " ခု\n";
  }
  
  else if (role === "SUPER") {
    message += "💰 *ဈေးနှုန်းများ*\n";
    message += "   📥 ဝယ်ယူဈေး : " + formatPrice(group.purchasePrice) + " Ks\n";
    message += "   🏷️ လက်ကား : " + formatPrice(group.wholesalePrice) + " Ks\n";
    message += "   🏷️ လက်လီ : " + formatPrice(group.retailPrice) + " Ks\n";
    message += "\n📦 *လက်ကျန်အသေးစိတ် (ဆိုင်အလိုက် / အရောင်အလိုက်)*\n";
    
    if (group.hasYP) {
      message += "🏪 *ယုဇနပလာဇာ*\n";
      for (var color in group.ypColors) {
        message += "   🎨 " + color + " : " + group.ypColors[color] + " ခု\n";
      }
    }
    if (group.hasGMP) {
      message += "🏪 *ဂမုန်းပွင့် (ကန်တော်လေး)*\n";
      for (var color in group.gmpColors) {
        message += "   🎨 " + color + " : " + group.gmpColors[color] + " ခု\n";
      }
    }
    message += "📊 စုစုပေါင်း : " + group.totalStock + " ခု\n";
  }
  
  message += "━━━━━━━━━━━━━━━━━━━━━";
  message += "\n🙏 Thanks!";
  
  return message;
}

function formatMultipleResultsWithButtons(products, searchText, role) {
  var TOTAL_LIMIT = 5;
  var displayProducts = [];
  var catalogData = getCatalogData();
  
  for (var i = 0; i < products.length && displayProducts.length < TOTAL_LIMIT; i++) {
    if (products[i].totalStock > 0) {
      var catalogItem = catalogData[products[i].skid];
      var displayName = products[i].name;
      if (catalogItem && catalogItem.displayName) {
        displayName = catalogItem.displayName;
      }
      displayProducts.push({
        skid: products[i].skid,
        name: displayName,
        price: products[i].retailPrice,
        stock: products[i].totalStock,
        category: products[i].category
      });
    }
  }
  
  var hasMore = false;
  for (var i = TOTAL_LIMIT; i < products.length; i++) {
    if (products[i].totalStock > 0) {
      hasMore = true;
      break;
    }
  }
  
  var message = "🔍 *'" + searchText + "'* နှင့် အတူဆုံး ပစ္စည်းများ\n";
  message += "━━━━━━━━━━━━━━━━━━━━━\n";
  
  for (var i = 0; i < displayProducts.length; i++) {
    var p = displayProducts[i];
    message += (i+1) + ". " + p.name + "\n";
    message += "   💰 " + formatPrice(p.price) + " Ks\n";
    if (role !== "SMILER" && role !== "RANGER") {
      message += "   📦 လက်ကျန် : " + p.stock + " ခု\n";
    }
    message += "\n";
  }
  
  if (hasMore) {
    message += "━━━━━━━━━━━━━━━━━━━━━\n";
    message += "⚠️ သင့်ရှာဖွေမှုနှင့် ကိုက်ညီသော ပစ္စည်း ထပ်ရှိပါသေးသည်။\n\n";
    message += "💡 *အကြံပြုချက်:* ပိုမိုတိကျစွာ ရိုက်ထည့်ပါက ရှာဖွေမှု အသေးစိတ် ကူညီနိုင်ပါသည်။\n\n";
  }
  
  message += "💡 အောက်ပါပစ္စည်းများကို နှိပ်၍ အသေးစိတ် ကြည့်ရှုနိုင်ပါသည်။";
  return { message: message, products: displayProducts };
}

// =====================================================
// ========== WELCOME WITH MENU ==========
// =====================================================

function sendWelcomeWithMenu(chatId, role) {
  var url = TELEGRAM_API_URL + "/sendMessage";
  
  var keyboard = {
    "inline_keyboard": [
      [
        {"text": "🏠 အစပြန်သွားမယ်", "callback_data": "MENU_HOME"}
      ],
      [
        {"text": "📱 Mobile ပစ္စည်းများ", "callback_data": "MENU_MOBILE"},
        {"text": "💄 Cosmetics ပစ္စည်းများ", "callback_data": "MENU_COSMETICS"}
      ],
      [
        {"text": "🔍 SKID ဖြင့် ရှာရန်", "callback_data": "HELP_SEARCH"},
        {"text": "🤖 AI Assistant", "callback_data": "MENU_AI"}
      ],
      [
        {"text": "🚀 Smile Portal", "web_app": {"url": "https://smile-cloud.github.io/smile-portal/"}},
        {"text": "📖 အသုံးပြုနည်း", "callback_data": "HELP"}
      ]
    ]
  };
  
  var payload = {
    'chat_id': String(chatId),
    'text': "✨ *Smile Mobile & Cosmetics Bot* မှ ကြိုဆိုပါတယ်။\n\n📌 *အောက်ပါ ခလုတ်များကို နှိပ်၍ လိုအပ်သော အချက်အလက်များ ရယူနိုင်ပါသည်။*",
    'parse_mode': 'Markdown',
    'reply_markup': JSON.stringify(keyboard)
  };
  
  var options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(payload)
  };
  UrlFetchApp.fetch(url, options);
}

// =====================================================
// ========== CALLBACK HANDLER ==========
// =====================================================

function handleCallbackQuery(callbackQuery) {
  var chatId = callbackQuery.message.chat.id;
  var data = callbackQuery.data;
  var role = getUserRole(chatId);
  
  Logger.log("🔔 Callback Data: " + data);
  
  if (data.startsWith("PROD_")) {
    var skid = data.replace("PROD_", "");
    var group = getProductGroup(skid);
    
    if (group && group.totalStock > 0) {
      var imageUrl = getProductImageUrl(skid);
      var caption = formatSingleProduct(group, role);
      
      if (imageUrl) {
        sendProductImageWithCaption(chatId, imageUrl, caption, group);
      } else {
        var reply = formatSingleProduct(group, role);
        sendSingleProductWithButton(chatId, reply, group);
      }
    } else if (group && group.totalStock === 0) {
      sendMessage(chatId, "❌ " + group.name + " သည် လက်ကျန်မရှိတော့ပါ။");
    } else {
      sendMessage(chatId, "❌ ပစ္စည်းကို ရှာမတွေ့ပါ။");
    }
  }
  
  else if (data === "CONTACT_MENU") {
    sendContactMenu(chatId);
  }
  else if (data === "CONTACT_PHONE") {
    sendPhoneContact(chatId);
  }
  else if (data === "CONTACT_VIBER") {
    sendViberContact(chatId);
  }
  else if (data === "CONTACT_TELEGRAM") {
    sendTelegramContact(chatId);
  }
  
  else if (data === "HELP_SEARCH") {
    sendMessage(chatId, "🔍 *ရှာဖွေနည်းလမ်းညွှန်*\n\n" +
                "📝 *သင်ရိုက်ထည့်နိုင်သော ပုံစံများ:*\n" +
                "• ပစ္စည်းအမည် - `Redmi Note 14`\n" +
                "• SKID - `MXN148128`\n" +
                "• Barcode နံပါတ် - `8901234567890`\n" +
                "• Barcode ပုံ - ပုံတစ်ပုံပို့ပါ\n\n" +
                "🤖 *AI Assistant ကိုလည်း မေးမြန်းနိုင်ပါသည်:*\n" +
                "• `/ai` - AI မုဒ်ဝင်ရန်\n" +
                "• ဥပမာ - `ဘယ်ဖုန်းက အကောင်းဆုံးလဲ`\n\n" +
                "🏠 `/start` - ပင်မစာမျက်နှာသို့");
  }
  
  else if (data === "START" || data === "MENU_HOME") {
    sendWelcomeWithMenu(chatId, role);
  }
  else if (data === "MENU_MOBILE") {
    sendMessage(chatId, "📱 *Mobile ပစ္စည်းများ*\n\n" +
                "သင်ရှာဖွေလိုသော ဖုန်းအမည် (သို့) SKID ကို ရိုက်ထည့်ပါ။\n\n" +
                "📝 ဥပမာ: `Redmi Note 14`, `MXN148128`\n\n" +
                "💡 /help - အကူအညီအတွက်");
  }
  else if (data === "MENU_COSMETICS") {
    sendMessage(chatId, "💄 *Cosmetics ပစ္စည်းများ*\n\n" +
                "သင်ရှာဖွေလိုသော အလှကုန်အမည် (သို့) SKID ကို ရိုက်ထည့်ပါ။\n\n" +
                "📝 ဥပမာ: `Foundation`, `1NMU03`\n\n" +
                "💡 /help - အကူအညီအတွက်");
  }
  else if (data === "MENU_AI") {
    sendMessage(chatId, "🤖 *AI Assistant Mode*\n\n" +
                "သင်၏မေးခွန်းကို မေးမြန်းနိုင်ပါသည်။\n\n" +
                "📝 *ဥပမာများ:*\n" +
                "• ဘယ်ဖုန်းက အကောင်းဆုံးလဲ\n" +
                "• Redmi Note 14 စျေးဘယ်လောက်လဲ\n" +
                "• လက်ကျန်ဘယ်နှစ်လုံးရှိလဲ\n\n" +
                "💡 `/start` - ပင်မစာမျက်နှာသို့");
  }
  else if (data === "HELP") {
    sendMessage(chatId, getHelpMessage(role));
  }
  
  var answerUrl = TELEGRAM_API_URL + "/answerCallbackQuery";
  var answerPayload = { 'callback_query_id': callbackQuery.id };
  UrlFetchApp.fetch(answerUrl, {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(answerPayload)
  });
}

// =====================================================
// ========== TELEGRAM REQUEST HANDLER ==========
// =====================================================

function doGet(e) {
  return ContentService.createTextOutput("SMILE Portal API is running. Use POST method for search.")
    .setMimeType(ContentService.MimeType.TEXT);
}

function doPost(e) {
  if (e && e.postData && e.postData.contents) {
    try {
      var data = JSON.parse(e.postData.contents);
      if (data.action === 'search') {
        return handleMiniAppSearch(data);
      }
      if (data.action === 'getPoints') {
        return handleGetPointsAPI(data);
      }
      if (data.action === 'getStamps') {
        return handleGetStampsAPI(data);
      }
      if (data.action === 'addPoints') {
        return handleAddPointsAPI(data);
      }
    } catch(err) {
      Logger.log("Web App API Error: " + err);
    }
  }
  return handleTelegramRequest(e);
}

function handleMiniAppSearch(data) {
  var query = data.query || "";
  var results = fuzzySearchProductsImproved(query);
  var output = [];
  for (var i = 0; i < results.length && output.length < 30; i++) {
    if (results[i].totalStock > 0) {
      output.push({
        skid: results[i].skid,
        name: results[i].name,
        price: results[i].retailPrice,
        stock: results[i].totalStock
      });
    }
  }
  return ContentService.createTextOutput(JSON.stringify(output)).setMimeType(ContentService.MimeType.JSON);
}

function handleGetPointsAPI(data) {
  var chatId = data.chatId;
  var balance = getPointsBalance(chatId);
  var spending = getCustomerTotalSpending(chatId);
  var tier = getCurrentCardTier(spending.total);
  
  return ContentService.createTextOutput(JSON.stringify({
    success: true,
    points: balance,
    tier: tier.color + " " + tier.tier + " Member",
    totalSpending: spending.total
  })).setMimeType(ContentService.MimeType.JSON);
}

function handleGetStampsAPI(data) {
  var chatId = data.chatId;
  var stampInfo = getCustomerStampInfo(chatId);
  
  return ContentService.createTextOutput(JSON.stringify({
    success: true,
    stamps: stampInfo.stampCount
  })).setMimeType(ContentService.MimeType.JSON);
}

function handleAddPointsAPI(data) {
  var phone = data.phone;
  var amount = data.amount;
  var category = data.category;
  var invoice = data.invoice;
  var cashierEmail = data.cashierEmail;
  
  var chatId = getChatIdByPhone(phone);
  
  if (!chatId) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      message: "❌ ဤဖုန်းနံပါတ်ဖြင့် စာရင်းသွင်းထားသူ မရှိပါ။"
    })).setMimeType(ContentService.MimeType.JSON);
  }
  
  var points = Math.floor(amount / 1000);
  var stamps = Math.floor(amount / 30000);
  
  addPointsToLedger(phone, chatId, points, amount, invoice);
  
  // Send notification to customer
  sendMessage(chatId, "🎉 *Points Added!*\n\n➕ +" + points + " points\n📮 +" + stamps + " stamps\n💰 " + formatPrice(amount) + " Ks");
  
  return ContentService.createTextOutput(JSON.stringify({
    success: true,
    message: "✅ Points " + points + " ထည့်သွင်းပြီးပါပြီ!\n📞 " + phone + "\n💰 " + formatPrice(amount) + " Ks\n📮 Stamps: " + stamps,
    chatId: chatId,
    points: points,
    stamps: stamps
  })).setMimeType(ContentService.MimeType.JSON);
}

function handleTelegramRequest(e) {
  try {
    var contents = JSON.parse(e.postData.contents);
    
    if (contents.callback_query) {
      handleCallbackQuery(contents.callback_query);
      return;
    }
    
    var message = contents.message;
    if (!message) return;
    
    var chatId = message.chat.id;
    var userName = message.from.first_name || "Customer";
    var role = getUserRole(chatId);
    var incomingText = message.text ? message.text.trim() : "";
    
    Logger.log("User: " + userName + " | Role: " + role + " | Text: " + incomingText);
    
    // Photo handler
    if (message.photo) {
      var photoId = message.photo[message.photo.length - 1].file_id;
      var barcodeReply = processBarcodePhoto(chatId, photoId, role);
      sendMessage(chatId, barcodeReply);
      return;
    }
    
    // /start command
    if (incomingText === '/start') {
      sendWelcomeWithMenu(chatId, role);
      return;
    }
    
    // /help command
    if (incomingText === '/help') {
      sendMessage(chatId, getHelpMessage(role));
      return;
    }
    
    // /ai command
    if (incomingText === '/ai') {
      var aiMsg = "🤖 *AI Assistant Mode*\n\n" +
                  "Send me any question about our products!\n\n" +
                  "📝 *Examples:*\n" +
                  "• ဘယ်ဖုန်းကအကောင်းဆုံးလဲ\n" +
                  "• Redmi Note 14 စျေးဘယ်လောက်လဲ\n" +
                  "• လက်ကျန်ဘယ်နှစ်လုံးရှိလဲ\n\n" +
                  "💡 `/start` - ပင်မစာမျက်နှာသို့";
      sendMessage(chatId, aiMsg);
      return;
    }
    
    // /points command
    if (incomingText === '/points') {
      var balance = getPointsBalance(chatId);
      var history = getPointsHistory(chatId, 5);
      var pointsReply = formatPointsMessage(balance, history);
      sendMessage(chatId, pointsReply);
      return;
    }
    
    // /stamp command
    if (incomingText === '/stamp') {
      var stampReply = formatStampStatus(chatId);
      sendMessage(chatId, stampReply);
      return;
    }
    
    // /status command
    if (incomingText === '/status') {
      var statusReply = formatCardStatus(chatId);
      sendMessage(chatId, statusReply);
      return;
    }
    
    // /points_register command
    if (incomingText === '/points_register') {
      if (checkPointsCustomerExists(chatId)) {
        sendMessage(chatId, "✅ သင်သည် စာရင်းသွင်းပြီးသားဖြစ်ပါသည်။\n📊 `/points` ဖြင့် စစ်ဆေးပါ။");
      } else {
        sendMessage(chatId, "📝 စာရင်းသွင်းရန် သင့်ဖုန်းနံပါတ်ကို ရိုက်ထည့်ပါ။\n📝 ဥပမာ: `09780001662`");
        setUserState(chatId, "AWAITING_POINTS_REG_PHONE");
      }
      return;
    }
    
    // /addpoints command (Cashier only - SUPER role)
if (incomingText.startsWith('/addpoints')) {
  if (role !== "SUPER") {
    sendMessage(chatId, "❌ သင့်တွင် ဤ Command ကိုသုံးရန် ခွင့်ပြုချက်မရှိပါ။");
  } else {
    var parts = incomingText.split(' ');
    var phone = parts[1];
    var amount = parseInt(parts[2]);
    var invoice = parts[3] || "MANUAL_" + new Date().getTime();
    
    if (!phone || !amount) {
      sendMessage(chatId, "❌ ပုံစံမှန်ကန်စွာ ရိုက်ထည့်ပါ။\n📝 ဥပမာ: `/addpoints 09780001662 50000 INV-001`");
    } else {
      var customerChatId = getChatIdByPhone(phone);
      if (!customerChatId) {
        sendMessage(chatId, "❌ ဤဖုန်းနံပါတ်ဖြင့် စာရင်းသွင်းထားသူ မရှိပါ။");
      } else {
        var points = Math.floor(amount / 1000);
        var stamps = Math.floor(amount / 30000);
        addPointsToLedger(phone, customerChatId, points, amount, invoice);
        sendMessage(chatId, "✅ Points " + points + " ထည့်သွင်းပြီးပါပြီ!\n📞 " + phone + "\n💰 " + formatPrice(amount) + " Ks\n📮 Stamps: +" + stamps);
        sendMessage(customerChatId, "🎉 *Points Added!*\n\n➕ +" + points + " points\n📮 +" + stamps + " stamps\n💰 " + formatPrice(amount) + " Ks\n\n📊 `/points` - သင့် Points ကိုစစ်ဆေးပါ");
      }
    }
  }
  return;
}
    
    // Handle phone registration
    if (getUserState(chatId) === "AWAITING_POINTS_REG_PHONE") {
      var phone = incomingText.trim();
      if (/^\d{10,11}$/.test(phone)) {
        var registerResult = registerPointsCustomer(chatId, phone, userName);
        sendMessage(chatId, registerResult.message);
        clearUserState(chatId);
      } else {
        sendMessage(chatId, "❌ ဖုန်းနံပါတ် မမှန်ပါ။ ကျေးဇူးပြု၍ ပြန်ရိုက်ပါ။\n📝 ဥပမာ: `09780001662`");
      }
      return;
    }
    
    // AI Assistant
    if (shouldUseAI(incomingText)) {
      sendChatAction(chatId, "typing");
      var aiReply = askAIAboutInventory(incomingText);
      sendMessage(chatId, aiReply);
      return;
    }
    
    // Product Search
    var result = getSmartReply(incomingText, role);
    
    if (result.type === 'single') {
      var replyText = formatSingleProduct(result.group, role);
      if (replyText) {
        sendSingleProductWithButton(chatId, replyText, result.group);
      }
    } 
    else if (result.type === 'multiple') {
      var formatted = formatMultipleResultsWithButtons(result.products, result.searchText, result.role);
      if (formatted.products.length > 0) {
        sendMessageWithProductButtons(chatId, formatted.message, formatted.products);
      } else {
        sendMessage(chatId, "❌ `" + result.searchText + "` နှင့်ဆက်စပ်သော လက်ကျန်ရှိပစ္စည်းမတွေ့ပါ။");
      }
    }
    else if (result.type === 'none') {
      sendMessage(chatId, result.message);
    }
    
  } catch (error) {
    Logger.log("Error: " + error.toString());
    sendMessage(chatId, "⚠️ စနစ်တွင်ယာယီအမှားတစ်ခုဖြစ်ပွားနေပါသည်။ ကျေးဇူးပြု၍နောက်မှထပ်မံကြိုးစားပါ။");
  }
}

// =====================================================
// ========== BARCODE SUPPORT ==========
// =====================================================

function downloadTelegramPhoto(fileId) {
  try {
    var getFileUrl = TELEGRAM_API_URL + "/getFile?file_id=" + fileId;
    var fileResponse = UrlFetchApp.fetch(getFileUrl);
    var fileData = JSON.parse(fileResponse.getContentText());
    if (!fileData.ok || !fileData.result.file_path) return null;
    var imageUrl = "https://api.telegram.org/file/bot" + BOT_TOKEN + "/" + fileData.result.file_path;
    var imageBlob = UrlFetchApp.fetch(imageUrl).getBlob();
    return imageBlob;
  } catch(e) {
    Logger.log("Download error: " + e.toString());
    return null;
  }
}

function processBarcodePhoto(chatId, photoFileId, role) {
  try {
    var imageBlob = downloadTelegramPhoto(photoFileId);
    if (!imageBlob) {
      return "📷 Barcode ပုံကိုဒေါင်းလုဒ်ဆွဲ၍မရပါ။\n\n💡 Barcode နံပါတ်ကိုတိုက်ရိုက်ရိုက်ထည့်ပါ။";
    }
    return "📷 Barcode ပုံကိုလက်ခံရရှိပါသည်။\n\n" +
           "💡 Barcode နံပါတ်ကိုတိုက်ရိုက်ရိုက်ထည့်ပါ။\n" +
           "📝 ဥပမာ: `8901234567890`";
  } catch(e) {
    Logger.log("Process barcode error: " + e.toString());
    return "📷 Barcode ဖတ်ရှုရာတွင်အမှားရှိပါသည်။\n\n💡 Barcode နံပါတ်ကိုတိုက်ရိုက်ရိုက်ထည့်ပါ။";
  }
}

// =====================================================
// ========== HELP MESSAGE ==========
// =====================================================

function getHelpMessage(role) {
  var msg = "📖 *အသုံးပြုနည်းအသေးစိတ်*\n\n";
  msg += "🔹 *ပစ္စည်းအမည်ဖြင့်ရှာရန်*\n";
  msg += "   ဥပမာ: `Redmi Note 14`, `Note 13 Pro`\n\n";
  msg += "🔹 *SKID ဖြင့်ရှာရန်*\n";
  msg += "   ဥပမာ: `MXN148128`\n\n";
  msg += "🔹 *Barcode ဖြင့်ရှာရန်*\n";
  msg += "   နံပါတ်ရိုက်ထည့်ပါ - `8901234567890`\n";
  msg += "   ပုံပို့ပါ - Barcode ပုံတစ်ပုံပို့ပါ\n\n";
  msg += "💎 *Points System*\n";
  msg += "   `/points` - Points လက်ကျန်စစ်ရန်\n";
  msg += "   `/points_register` - စာရင်းသွင်းရန်\n";
  msg += "   `/stamp` - Stamps စစ်ရန်\n";
  msg += "   `/status` - Card အဆင့်စစ်ရန်\n\n";
  msg += "🤖 *AI Assistant*\n";
  msg += "   `/ai` - AI ကိုမေးမြန်းရန်\n";
  msg += "   ဥပမာ: `ဘယ်ဖုန်းကအကောင်းဆုံးလဲ`\n\n";
  msg += "🚀 *Mini App*\n";
  msg += "   Smile Portal ခလုတ်ကိုနှိပ်၍ Web App သုံးနိုင်ပါသည်။\n\n";
  msg += "🏠 `/start` - ပင်မစာမျက်နှာသို့\n";
  msg += "👤 *သင်၏ Role*: `" + role + "`";
  return msg;
}

// =====================================================
// ========== WEBHOOK MANAGEMENT ==========
// =====================================================

function forceSetWebhook() {
  var CORRECT_URL = "https://script.google.com/macros/s/AKfycbyOaZWRwJaH6q-z1KNMmraOpMm_0HVCO4Vs-I0B_DTTShnsK59a3J-DTnY_FhFGb1A/exec";
  
  var deleteUrl = TELEGRAM_API_URL + '/deleteWebhook';
  UrlFetchApp.fetch(deleteUrl);
  
  var setUrl = TELEGRAM_API_URL + '/setWebhook?url=' + encodeURIComponent(CORRECT_URL);
  UrlFetchApp.fetch(setUrl);
  
  SpreadsheetApp.getActiveSpreadsheet().toast('✅ Webhook Set!', 'Telegram Bot', 5);
}

function getWebhookInfo() {
  var url = TELEGRAM_API_URL + '/getWebhookInfo';
  var response = UrlFetchApp.fetch(url);
  Logger.log(response.getContentText());
}

function testSendMessage() {
  sendMessage(ADMIN_CHAT_ID, '🧪 *Test Message* from SMILE Bot!\n\nBot is working correctly. ✅');
}
