import { Telegraf } from 'telegraf';
import express from 'express';

// --- SERVER FOR 24/7 UPTIME ---
const app = express();
app.get('/', (req, res) => res.send('Bot is Alive!'));
app.listen(3000, () => console.log('Web server is running for uptime monitoring.'));

// --- BOT CONFIGURATION ---
const bot ="8639726145:AAEwp4zpsLi4E8cpXQaoCEgwrr1yfCfkoiE"

interface SessionData {
  members: number;
  timerInterval: NodeJS.Timeout | null;
  messageId: number;
  chatId: number;
  joinedUsers: Set<number>;
}

let sessionData: SessionData = {
  members: 0,
  timerInterval: null,
  messageId: 0,
  chatId: 0,
  joinedUsers: new Set<number>()
};

// --- HELPER FUNCTION: START TIMER ---
const startTimer = (ctx: any, durationInSeconds: number, title: string) => {
  // Clear any existing timer before starting a new one
  if (sessionData.timerInterval) {
    clearInterval(sessionData.timerInterval);
  }

  // Reset session data
  let secondsRemaining = durationInSeconds;
  sessionData.members = 0;
  sessionData.joinedUsers.clear();
  sessionData.chatId = ctx.chat.id;

  ctx.reply(`${title} started! ⏱️`).then((message: any) => {
    sessionData.messageId = message.message_id;

    sessionData.timerInterval = setInterval(() => {
      secondsRemaining--;

      // Time calculation
      const hours = Math.floor(secondsRemaining / 3600);
      const minutes = Math.floor((secondsRemaining % 3600) / 60);
      const seconds = secondsRemaining % 60;

      // Formatting display: show hours only if it's a long session
      const timeDisplay = hours > 0 
        ? `${hours}h ${minutes}m ${seconds}s` 
        : `${minutes}m ${seconds}s`;

      const timerText = `${title} Active ⏱️\n\nTime remaining: ${timeDisplay}\nTotal members: ${sessionData.members}`;

      // Update message in Telegram
      bot.telegram.editMessageText(sessionData.chatId, sessionData.messageId, undefined, timerText, {
        reply_markup: {
          inline_keyboard: [[{ text: 'Join Session 📖', callback_data: 'join' }]]
        }
      }).catch(() => {
        // Silently catch errors from Telegram API rate limits
      });

      // End of session logic
      if (secondsRemaining <= 0) {
        if (sessionData.timerInterval) clearInterval(sessionData.timerInterval);
        sessionData.timerInterval = null;
        ctx.telegram.editMessageText(sessionData.chatId, sessionData.messageId, undefined, `${title} ended! ✅\nTotal members: ${sessionData.members}`);
      }
    }, 1000); // 1 second update
  });
};

// --- COMMANDS ---

// Study Command (6 Hours)
bot.command('study', (ctx) => startTimer(ctx, 6 * 60 * 60, 'Study Session'));

// Break Command (20 Minutes)
bot.command('break', (ctx) => startTimer(ctx, 20 * 60, 'Break Session'));

// Cancel Command
bot.command('cancel', (ctx) => {
  if (sessionData.timerInterval) {
    clearInterval(sessionData.timerInterval);
    sessionData.timerInterval = null;
    ctx.reply('The session has been cancelled. 🛑');
  } else {
    ctx.reply('There is no active session to cancel.');
  }
});

// --- ACTIONS (BUTTONS) ---

bot.action('join', (ctx) => {
  const userId = ctx.from?.id;

  if (userId) {
    if (!sessionData.joinedUsers.has(userId)) {
      sessionData.joinedUsers.add(userId);
      sessionData.members++;
      ctx.answerCbQuery('You have successfully joined the session!');
    } else {
      ctx.answerCbQuery('You are already a member of this session!');
    }
  }
});

// --- LAUNCH BOT ---
bot.launch().then(() => {
  console.log('Telegram Bot is running smoothly!');
});

// Enable graceful stop
process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));

