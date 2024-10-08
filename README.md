const { Telegraf } = require('telegraf');
const { Connection, PublicKey, LAMPORTS_PER_SOL } = require('@solana/web3.js');
const Alchemy = require('alchemy-sdk');
const Redis = require('redis');
const fetch = require('node-fetch');

// Configuration and initializing variables
const BOT_TOKEN = '8086588949:AAHzO1j5J4WAmOZe0CXB4XW3BahhLMtSma0';
const ALCHEMY_API_KEY = 'hwSryq9_VnJJR2qQ1eIqmugwzjri3eJI';
const SOLANA_WALLET = 'H94k8nyQVfiybxvG13kGV6hY9X9eDdZgsA6NL5VrqfS6';
const CHAT_ID = '-1002467347399';
const BASE_FEE = 0.005; // Base transaction fee
const REFERRAL_FEE = 0.004; // Fee for referred users
const REFERRER_COMMISSION = 0.0005; // Commission given to referrer

// Initialize Telegram bot, Alchemy API, Solana connection, and Redis
const bot = new Telegraf(BOT_TOKEN);
const connection = new Connection('https://api.mainnet-beta.solana.com');
const alchemy = new Alchemy({ apiKey: ALCHEMY_API_KEY, network: Alchemy.SolanaNetwork.MAINNET });
const redisClient = Redis.createClient();
redisClient.connect();

// Storing user information and transaction logs
let users = {};
let logs = {};

// Function to log all transactions
async function logTransaction(chatId, action, status, details) {
  if (!logs[chatId]) logs[chatId] = [];
  logs[chatId].push({
    action,
    status,
    details,
    timestamp: new Date().toISOString()
  });
  await redisClient.hSet(chatId, 'logs', JSON.stringify(logs[chatId]));
}

// Function to generate a wallet for each user
async function generateUserWallet(chatId) {
  const wallet = new PublicKey();
  users[chatId] = {
    wallet: wallet.toBase58(),
    balance: 0,
    tokens: {},
    referral: null,
    totalTrades: 0,
    totalProfit: 0
  };
  await redisClient.hSet(chatId, users[chatId]);
  await logTransaction(chatId, 'generateWallet', 'success', `Generated wallet: ${wallet.toBase58()}`);
  return wallet.toBase58();
}

// Function to check user balance
async function checkUserBalance(chatId) {
  const user = await redisClient.hGetAll(chatId);
  return user && user.balance >= 1;
}

// Function to calculate transaction fees based on referral status
function calculateTransactionFee(chatId) {
  if (users[chatId]?.referral) {
    return [REFERRAL_FEE, REFERRER_COMMISSION];
  }
  return [BASE_FEE, 0];
}

// Function to process the transaction and apply fees
async function processTransaction(chatId, amount) {
  const [fee, referrerCommission] = calculateTransactionFee(chatId);
  const netAmount = amount - fee;

  const referrerId = users[chatId].referral;
  if (referrerId) {
    const referrerWallet = users[referrerId].wallet;
    await connection.requestAirdrop(new PublicKey(referrerWallet), referrerCommission * LAMPORTS_PER_SOL);
    await logTransaction(referrerId, 'referralEarnings', 'success', `Earned referral commission: ${referrerCommission} SOL`);
  }

  // Send transaction fee to the main wallet
  await connection.requestAirdrop(new PublicKey(SOLANA_WALLET), fee * LAMPORTS_PER_SOL);
  await logTransaction(chatId, 'transactionFee', 'success', `Charged fee: ${fee} SOL`);
  return netAmount;
}

// Create the purchase object to save in the database
async function createPurchaseObject(chatId, tokenAddress, amount, status) {
  const purchase = {
    chatId,
    tokenAddress,
    amount,
    status,
    date: new Date().toISOString()
  };
  await redisClient.hSet(chatId, 'purchase', JSON.stringify(purchase));
  await logTransaction(chatId, 'purchase', status, `Saved purchase: ${amount} of ${tokenAddress}`);
}

// Insert the purchase information into the database
async function savePurchaseToDatabase(chatId, tokenAddress, amount, status) {
  try {
    await createPurchaseObject(chatId, tokenAddress, amount, status);
    await bot.telegram.sendMessage(chatId, `💾 Your purchase of ${amount} ${tokenAddress} was saved successfully.`);
  } catch (error) {
    await bot.telegram.sendMessage(chatId, `❌ Error saving purchase: ${error.message}`);
  }
}

// Function to handle the buy flow for tokens
let userInputState = {};  // State to handle user input across various stages

// Handler for the /buy command
bot.command('buy', async (ctx) => {
  const chatId = ctx.message.chat.id;
  const tokenAddress = ctx.message.text.split(' ')[1];

  if (!await checkUserBalance(chatId)) {
    await ctx.reply(`⚠️ You need to deposit at least 1 SOL to use this feature.\nPlease deposit SOL into your wallet.`);
    await logTransaction(chatId, 'buy', 'error', 'User has insufficient balance');
    return;
  }

  const tokenInfo = await alchemy.core.getTokenInfo(tokenAddress);
  const tokenMessage = `📊 Token Info:\n🔹 Price: ${tokenInfo.price}\n🔹 Liquidity: ${tokenInfo.liquidity}\n🔹 Market Cap: ${tokenInfo.market_cap}\n🔹 24h Volume: ${tokenInfo.volume24h}\n🔹 Holders: ${tokenInfo.holdersCount}\n💰 Your current balance: ${users[chatId].balance} SOL`;

  await ctx.reply(tokenMessage);
  await logTransaction(chatId, 'buy', 'info', `User requested token info: ${tokenAddress}`);

  // Display buttons for buying different amounts of tokens
  const buttons = [
    [{ text: 'Buy 0.2 SOL', callback_data: `buy_0.2_${tokenAddress}` }],
    [{ text: 'Buy 1 SOL', callback_data: `buy_1_${tokenAddress}` }],
    [{ text: 'Buy Custom Amount', callback_data: `buy_custom_${tokenAddress}` }]
  ];

  await ctx.reply('🛒 Select the amount to buy:', {
    reply_markup: { inline_keyboard: buttons }
  });
});

// Process the purchase transaction
// Save the purchase information into the database
async function processBuyTransaction(chatId, tokenAddress, solAmount) {
  const userBalance = users[chatId].balance;

  if (userBalance >= solAmount) {
    const netAmount = await processTransaction(chatId, solAmount);
    users[chatId].balance -= solAmount;
    users[chatId].tokens[tokenAddress] = (users[chatId].tokens[tokenAddress] || 0) + netAmount;
    await redisClient.hSet(chatId, 'balance', users[chatId].balance);
    await redisClient.hSet(chatId, 'tokens', JSON.stringify(users[chatId].tokens));

    await savePurchaseToDatabase(chatId, tokenAddress, solAmount, 'success');
    await bot.telegram.sendMessage(chatId, `✅ Transaction successful! You have purchased ${netAmount} worth of ${tokenAddress}.\n📊 Your new balance is ${users[chatId].balance} SOL.`);
    await logTransaction(chatId, 'buy', 'success', `Bought ${netAmount} SOL worth of ${tokenAddress}`);
  } else {
    await savePurchaseToDatabase(chatId, tokenAddress, solAmount, 'failed');
    await bot.telegram.sendMessage(chatId, `⚠️ Insufficient balance. You have ${userBalance} SOL but you need ${solAmount} SOL to complete this transaction.`);
    await logTransaction(chatId, 'buy', 'error', 'Insufficient balance for purchase');
  }
}

// Handles user interaction with the Buy buttons
bot.action(/^buy_/, async (ctx) => {
  const chatId = ctx.from.id;
  const [_, amount, tokenAddress] = ctx.match.input.split('_');

  if (amount === 'custom') {
    await ctx.reply('📨 Please send the custom amount you want to buy:');
    bot.on('message', async (msg) => {
      const customAmount = parseFloat(msg.text);
      await processBuyTransaction(chatId, tokenAddress, customAmount);
    });
  } else {
    await processBuyTransaction(chatId, tokenAddress, parseFloat(amount));
  }
});

// Handler for the /sell command
bot.command('sell', async (ctx) => {
  const chatId = ctx.message.chat.id;
  const tokenAddress = ctx.message.text.split(' ')[1];

  if (!users[chatId].tokens[tokenAddress]) {
    await ctx.reply(`⚠️ You don't own any tokens of this type.`);
    await logTransaction(chatId, 'sell', 'error', `User tried to sell a token they don't own`);
    return;
  }

  // Display options to sell different percentages or amounts
  await ctx.reply('🔄 Select the percentage to sell:', {
    reply_markup: {
      inline_keyboard: [
        [{ text: 'Sell 50%', callback_data: `sell_50_${tokenAddress}` }],
        [{ text: 'Sell 100%', callback_data: `sell_100_${tokenAddress}` }],
        [{ text: 'Sell Custom Amount', callback_data: `sell_custom_${tokenAddress}` }]
      ]
    }
  });
});

// Function to process the sale transaction
// Save the sale information into the database
async function processSellTransaction(chatId, tokenAddress, amount) {
  const userTokenBalance = users[chatId].tokens[tokenAddress];

  if (amount <= userTokenBalance) {
    const netAmount = await processTransaction(chatId, amount);
    users[chatId].tokens[tokenAddress] -= amount;
    users[chatId].balance += netAmount;
    await redisClient.hSet(chatId, 'balance', users[chatId].balance);
    await redisClient.hSet(chatId, 'tokens', JSON.stringify(users[chatId].tokens));

    await savePurchaseToDatabase(chatId, tokenAddress, amount, 'success');
    await bot.telegram.sendMessage(chatId, `✅ Transaction successful! You have sold ${amount} tokens for ${netAmount} SOL.\n📊 Your new balance is ${users[chatId].balance} SOL.`);
    await logTransaction(chatId, 'sell', 'success', `Sold ${amount} tokens for ${netAmount} SOL`);
  } else {
    await savePurchaseToDatabase(chatId, tokenAddress, amount, 'failed');
    await bot.telegram.sendMessage(chatId, `⚠️ Insufficient token balance. You only have ${userTokenBalance} tokens, but you tried to sell ${amount}.`);
    await logTransaction(chatId, 'sell', 'error', 'Insufficient token balance');
  }
}

// Handles user interaction with the Sell buttons
bot.action(/^sell_/, async (ctx) => {
  const chatId = ctx.from.id;
  const [_, percentage, tokenAddress] = ctx.match.input.split('_');

  const userTokenBalance = users[chatId].tokens[tokenAddress];
  let sellAmount = 0;

  if (percentage === 'custom') {
    await ctx.reply('📨 Please send the custom amount you want to sell:');
    bot.on('message', async (msg) => {
      sellAmount = parseFloat(msg.text);
      await processSellTransaction(chatId, tokenAddress, sellAmount);
    });
  } else {
    sellAmount = (percentage === '50') ? userTokenBalance / 2 : userTokenBalance;
    await processSellTransaction(chatId, tokenAddress, sellAmount);
  }
});

// Function to handle withdrawals
// Handler for the /withdraw command
bot.command('withdraw', async (ctx) => {
  const chatId = ctx.message.chat.id;
  const amount = parseFloat(ctx.message.text.split(' ')[1]);

  if (users[chatId].balance >= amount) {
    users[chatId].balance -= amount;
    await redisClient.hSet(chatId, 'balance', users[chatId].balance);
    await connection.requestAirdrop(new PublicKey(users[chatId].wallet), amount * LAMPORTS_PER_SOL);
    await ctx.reply(`✅ Withdrawal successful! You withdrew ${amount} SOL.\n📊 Your new balance is ${users[chatId].balance} SOL.`);
    await logTransaction(chatId, 'withdraw', 'success', `User withdrew ${amount} SOL`);
  } else {
    await ctx.reply(`⚠️ Insufficient balance for withdrawal. You have ${users[chatId].balance} SOL, but you tried to withdraw ${amount} SOL.`);
    await logTransaction(chatId, 'withdraw', 'error', 'Insufficient balance for withdrawal');
  }
});

// Function to generate referral link
// Handler for the /referral command
bot.command('referral', async (ctx) => {
  const chatId = ctx.message.chat.id;
  const referralLink = `https://t.me/LynexisBot?start=${chatId}`;
  await ctx.reply(`🎉 Share this link to earn rewards: ${referralLink}`);
  await logTransaction(chatId, 'referral', 'info', 'User generated referral link');
});

// Function to handle the portfolio viewing
// Handler for the /portfolio command
bot.command('portfolio', async (ctx) => {
  const chatId = ctx.message.chat.id;
  if (Object.keys(users[chatId].tokens).length === 0) {
    await ctx.reply(`⚠️ You don't own any tokens.`);
    await logTransaction(chatId, 'portfolio', 'info', 'User has no tokens');
    return;
  }

  let portfolioMessage = `📊 Your portfolio contains the following tokens:\n`;
  for (const [token, amount] of Object.entries(users[chatId].tokens)) {
    portfolioMessage += `🔹 ${token}: ${amount} tokens\n`;
  }

  await ctx.reply(portfolioMessage);
  await logTransaction(chatId, 'portfolio', 'info', 'User viewed portfolio');
});

// Function to handle viewing user statistics
// Handler for the /stats command
bot.command('stats', async (ctx) => {
  const chatId = ctx.message.chat.id;
  const userStats = {
    totalTrades: users[chatId].totalTrades || 0,
    totalProfit: users[chatId].totalProfit || 0
  };

  await ctx.reply(`📊 Total trades: ${userStats.totalTrades}\n💰 Total profit: ${userStats.totalProfit} SOL`);
  await logTransaction(chatId, 'stats', 'info', 'User requested stats');
});

// Launch the bot and handle errors
bot.launch().then(() => {
  console.log('Bot is running...');
}).catch((error) => {
  console.error('Error launching bot:', error);
  process.exit(1); // Exit process if bot fails to launch
});
