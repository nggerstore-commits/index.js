require('dotenv').config();
const { 
  makeWASocket, 
  useMultiFileAuthState, 
  DisconnectReason, 
  fetchLatestBaileysVersion 
} = require('@whiskeysockets/baileys');
const fs = require('fs');
const axios = require('axios');

// ═══════════════════════════════════════
// CONFIG
// ═══════════════════════════════════════
const config = {
  antiLink: true,
  antiToxic: true,
  antiDelete: false,
  welcome: true
};

// Kata-kata toxic yang akan dihapus
const toxicWords = [
  'anjing', 'babi', 'kontol', 'memek', 'ngentot', 'tolol', 'bodoh', 
  'goblok', 'sokane', 'setan', 'iblis', 'asu', 'bangsat', 'g\0',
  'fuck', 'shit', 'damn', 'bitch', 'asshole', 'idiot', 'stupid'
];

async function startBot() {
  const { version } = await fetchLatestBaileysVersion();
  const { state, saveCreds } = await useMultiFileAuthState('session');

  const sock = makeWASocket({
    version,
    printQRInTerminal: true,
    auth: state,
    logger: { info: () => {}, error: () => {} }
  });

  sock.ev.on('creds.update', saveCreds);

  sock.ev.on('connection.update', (update) => {
    const { connection, lastDisconnect, qr } = update;
    if (qr) {
      console.log('\n📱 SCAN QR DI BAWAH:\n');
    }
    if (connection === 'close') {
      const reason = lastDisconnect?.error?.output?.statusCode;
      if (reason !== DisconnectReason.loggedOut) startBot();
    }
    if (connection === 'open') {
      console.log('✅ BOT ONLINE - SIAP DIGUNAKAN!');
    }
  });

  // Welcome message
  sock.ev.on('group-participants.update', async ({ id, participants, action }) => {
    if (!config.welcome) return;
    
    const group = await sock.groupMetadata(id);
    const groupName = group.subject;
    
    if (action === 'add') {
      for (const participant of participants) {
        const ppuser = participant.split('@')[0];
        await sock.sendMessage(id, {
          text: `👋 Halo @${ppuser}!\n\nSelamat datang di group *${groupName}*\n\n📜-rules:\n1. Jangan spam link\n2. Jangan toxic\n3. Enjoy!`,
          mentions: [participant]
        });
      }
    } else if (action === 'remove') {
      for (const participant of participants) {
        const ppuser = participant.split('@')[0];
        await sock.sendMessage(id, {
          text: `👋 Bye @${ppuser}!`,
          mentions: [participant]
        });
      }
    }
  });

  // Message handler
  sock.ev.on('messages.upsert', async ({ messages }) => {
    for (const msg of messages) {
      if (!msg.message || msg.key.fromMe) continue;

      const chatId = msg.key.remoteJid;
      const sender = msg.key.participant;
      const isGroup = chatId.endsWith('@g.us');
      
      const text = msg.message?.conversation || msg.message?.extendedTextMessage?.text || '';
      
      // ═══ AUTO MODERASI (Anti Link & Toxic) ═══
      if (isGroup && text) {
        await moderasiGroup(sock, msg, chatId, sender, text);
      }

      if (!text.startsWith('!')) continue;

      const args = text.slice(1).trim().split(' ');
      const cmd = args.shift().toLowerCase();
      const query = args.join(' ');

      console.log(`\n📩 [${isGroup ? 'GROUP' : 'PV'}] !${cmd} ${query}`);

      // ═══ FITUR: STIKER ═══
      if (cmd === 'stiker') {
        await buatStiker(sock, msg, chatId);
      }
      // ═══ FITUR: CARI LAGU ═══
      else if (cmd === 'lagu' || cmd === 'music') {
        await cariLagu(sock, msg, chatId, query);
      }
      // ═══ FITUR: MENU ═══
      else if (cmd === 'menu' || cmd === 'help' || cmd === 'cmd') {
        await showMenu(sock, msg, chatId);
      }
      // ═══ FITUR: KICK ═══
      else if (cmd === 'kick' && isGroup) {
        await kickMember(sock, msg, chatId, sender);
      }
      // ═══ FITUR: PROMOTE ═══
      else if (cmd === 'promote' && isGroup) {
        await promoteMember(sock, msg, chatId, sender);
      }
      // ═══ FITUR: DEMOTE ═══
      else if (cmd === 'demote' && isGroup) {
        await demoteMember(sock, msg, chatId, sender);
      }
      // ═══ FITUR: INFO GRUP ═══
      else if (cmd === 'groupinfo' || cmd === 'ginfo') {
        await infoGroup(sock, msg, chatId);
      }
      // ═══ FITUR: SET WELCOME ═══
      else if (cmd === 'welcome' && isGroup) {
        await setWelcome(sock, msg, chatId, query);
      }
      // ═══ FITUR: ANTI LINK ON/OFF ═══
      else if (cmd === 'antilink' && isGroup) {
        await toggleAntiLink(sock, msg, chatId, query);
      }
      // ═══ FITUR: BAN ═══
      else if (cmd === 'ban' && isGroup) {
        await banMember(sock, msg, chatId, sender);
      }
      // Unknown command
      else {
        await sock.sendMessage(chatId, {
          text: `❓ Command "!${cmd}" tidak ada.\n\nKetik *!menu* untuk melihat menu.`
        }, { quoted: msg });
      }
    }
  });
}

// ═══════════════════════════════════════
// FITUR: BUAT STIKER DARI FOTO
// ═══════════════════════════════════════
async function buatStiker(sock, msg, chatId) {
  try {
    const quoted = msg.message?.extendedTextMessage?.contextInfo?.quotedMessage;
    const img = quoted?.imageMessage || msg.message?.imageMessage;

    if (!img) {
      await sock.sendMessage(chatId, {
        text: `📎 *CARA BUAT STIKER:*\n\n` +
              `1. Kirim/unggah foto\n` +
              `2. Reply foto tersebut\n` +
              `3. Ketik: !stiker\n\n` +
              `✨ Stiker akan jadi dalam beberapa detik!`
      }, { quoted: msg });
      return;
    }

    await sock.sendMessage(chatId, { text: '⏳ Membuat stiker...' }, { quoted: msg });

    const buffer = await sock.downloadMediaMessage(msg.messages[0]);
    if (!buffer) throw new Error('Gagal download');
    
    const file = `temp_${Date.now()}.jpg`;
    fs.writeFileSync(file, buffer);

    await sock.sendMessage(chatId, { sticker: { url: file } }, { quoted: msg });
    
    // Cleanup
    setTimeout(() => {
      try { fs.unlinkSync(file); } catch {}
    }, 5000);
    
  } catch (e) {
    console.error('Stiker error:', e);
    await sock.sendMessage(chatId, { 
      text: '❌ Gagal buat stiker.\n\nPastikan foto jelas dan ukuran tidak terlalu besar.' 
    }, { quoted: msg });
  }
}

// ═══════════════════════════════════════
// FITUR: CARI LAGU + PENCIATA & ARTWORK
// ═══════════════════════════════════════
async function cariLagu(sock, msg, chatId, query) {
  if (!query) {
    await sock.sendMessage(chatId, {
      text: `🎵 *CARI LAGU*\n\n` +
            `Usage: !lagu <judul>\n` +
            `Contoh: !lagu melukis patah\n` +
            `       !lagu ada apa dengan cinta`
    }, { quoted: msg });
    return;
  }

  try {
    await sock.sendMessage(chatId, { text: `🔍 Mencari "${query}"...` }, { quoted: msg });

    // Cari di iTunes
    const res = await axios.get(
      `https://itunes.apple.com/search?term=${encodeURIComponent(query)}&limit=3`
    );

    if (res.data.resultCount === 0) {
      // Coba cari di Chord/API lain
      const fallback = await axios.get(
        `https://api.genius.com/search?q=${encodeURIComponent(query)}`,
        { headers: { Authorization: ' Bearer ' + process.env.GENIUS_TOKEN } }
      ).catch(() => ({ data: { response: { hits: [] } } }));

      if (fallback.data.response.hits.length > 0) {
        const hit = fallback.data.response.hits[0];
        await sock.sendMessage(chatId, {
          text: `🎵 *${hit.result.title}*\n` +
                `🎤 ${hit.result.primary_artist.name}\n\n` +
                `🔗 ${hit.result.url}`
        }, { quoted: msg });
        return;
      }

      await sock.sendMessage(chatId, {
        text: `❌ Lagu "${query}" tidak ketemu.\n\nCoba dengan nama lain atau lebih spesifik.`
      }, { quoted: msg });
      return;
    }

    // Tampilkan hasil dari iTunes
    for (const t of res.data.results) {
      const durasi = Math.floor(t.trackTimeMillis / 60000) + ':' + 
        String(Math.floor((t.trackTimeMillis % 60000) / 1000)).padStart(2, '0');

      let txt = `🎵 *${t.trackName}*\n\n`;
      txt += `🎤 *Pencipta:* ${t.artistName}\n`;
      txt += `💿 Album: ${t.collectionName || 'Single'}\n`;
      txt += `⏱️ Durasi: ${durasi}\n`;
      txt += `🎧 Genre: ${t.primaryGenreName}\n`;
      if (t.trackExplicitness) txt += `⚠️ Explicit: ${t.trackExplicitness}\n`;
      if (t.releaseDate) txt += `📅 Rilis: ${t.releaseDate.split('-')[0]}\n`;
      txt += `\n🔗 ${t.trackViewUrl}`;

      // Kirim artwork
      if (t.artworkUrl100) {
        const imgUrl = t.artworkUrl100.replace('100x100', '600x600');
        await sock.sendMessage(chatId, {
          image: { url: imgUrl },
          caption: `🎵 ${t.trackName}`
        }, { quoted: msg });
      }

      await sock.sendMessage(chatId, { text: txt }, { quoted: msg });

      // Hanya kirim 1 hasil utama
      break;
    }

  } catch (e) {
    console.error('Music error:', e);
    await sock.sendMessage(chatId, { text: '❌ Error cari lagu. Cobain lagi nanti.' }, { quoted: msg });
  }
}

// ═══════════════════════════════════════
// FITUR: MENU
// ═══════════════════════════════════════
async function showMenu(sock, msg, chatId) {
  const menuText = `🤖 *BOT WHATSAPP MENU*

╔══════════════════════╗
║     📌 FITUR UTAMA    ║
╠══════════════════════╣
║ 1. !stiker           ║
║    Ubah foto ke stiker║
║                      ║
║ 2. !lagu <judul>     ║
║    Cari info + pencipta║
╚══════════════════════╝

╔══════════════════════╗
║    📛 ADMIN GRUP     ║
╠══════════════════════╣
║ ▶ !kick @member      ║
║   Kick member        ║
║                      ║
║ ▶ !promote @member   ║
║   Jadikan admin      ║
║                      ║
║ ▶ !demote @admin    ║
║   Turunkan jadi member║
║                      ║
║ ▶ !groupinfo        ║
║   Info grup          ║
╚══════════════════════╝

╔══════════════════════╗
║    🛡️ AUTO MODERASI  ║
╠══════════════════════╣
║ ▶ Anti Link (auto)  ║
║ ▶ Anti Toxic (auto) ║
║ ▶ Anti Delete       ║
║ ▶ Welcome Message  ║
╚══════════════════════╝

Ketik *!menu* untuk menampilkan ini.`;

  await sock.sendMessage(chatId, { text: menuText }, { quoted: msg });
}

// ═══════════════════════════════════════
// FITUR: MODERASI GROUP (Anti Link & Toxic)
// ═══════════════════════════════════════
async function moderasiGroup(sock, msg, chatId, sender, text) {
  const senderNum = sender.split('@')[0];
  
  // Cek admin
  const group = await sock.groupMetadata(chatId);
  const isAdmin = group.participants.find(p => p.id === sender)?.admin;
  
  if (isAdmin) return;

  // Anti Link
  if (config.antiLink) {
    const linkPattern = /chat\.whatsapp\.com|wa\.me|bit\.ly|tinyurl|goo\.gl|http|https:\/\//i;
    if (linkPattern.test(text)) {
      await sock.sendMessage(chatId, {
        delete: { remoteJid: chatId, fromMe: false, id: msg.key.id, participant: sender }
      });
      await sock.sendMessage(chatId, {
        text: `⛔ @${senderNum} Dilarang Kirim Link!\n(Warni 1x)`,
        mentions: [sender]
      });
      return;
    }
  }

  // Anti Toxic
  if (config.antiToxic) {
    const textLower = text.toLowerCase();
    for (const word of toxicWords) {
      if (textLower.includes(word)) {
        await sock.sendMessage(chatId, {
          delete: { remoteJid: chatId, fromMe: false, id: msg.key.id, participant: sender }
        });
        await sock.sendMessage(chatId, {
          text: `⛔ @${senderNum} Jangan Toxic!\n(Warni 1x)`,
          mentions: [sender]
        });
        return;
      }
    }
  }
}

// ═══════════════════════════════════════
// FITUR: KICK MEMBER
// ═══════════════════════════════════════
async function kickMember(sock, msg, chatId, sender) {
  try {
    const group = await sock.groupMetadata(chatId);
    const isAdmin = group.participants.find(p => p.id === sender)?.admin;

    if (!isAdmin) {
      await sock.sendMessage(chatId, { text: '❌ Hanya admin yang bisa kick member!' }, { quoted: msg });
      return;
    }

    const mentioned = msg.message?.extendedTextMessage?.contextInfo?.mentionedJid?.[0];
    const quoted = msg.message?.extendedTextMessage?.contextInfo?.quotedParticipant;
    const target = mentioned || quoted;

    if (target) {
      await sock.groupParticipantsUpdate(chatId, [target], 'remove');
      await sock.sendMessage(chatId, {
        text: `👋 @${target.split('@')[0]} telah dikick dari grup!`,
        mentions: [target]
      });
    } else {
      await sock.sendMessage(chatId, { text: 'Usage: !kick @member' }, { quoted: msg });
    }
  } catch (e) {
    await sock.sendMessage(chatId, { text: '❌ Gagal kick member.' }, { quoted: msg });
  }
}

// ═══════════════════════════════════════
// FITUR: PROMOTE MEMBER
// ═══════════════════════════════════════
async function promoteMember(sock, msg, chatId, sender) {
  try {
    const group = await sock.groupMetadata(chatId);
    const isAdmin = group.participants.find(p => p.id === sender)?.admin;

    if (!isAdmin) {
      await sock.sendMessage(chatId, { text: '❌ Hanya admin yang bisa promote member!' }, { quoted: msg });
      return;
    }

    const mentioned = msg.message?.extendedTextMessage?.contextInfo?.mentionedJid?.[0];
    const quoted = msg.message?.extendedTextMessage?.contextInfo?.quotedParticipant;
    const target = mentioned || quoted;

    if (target) {
      await sock.groupParticipantsUpdate(chatId, [target], 'promote');
      await sock.sendMessage(chatId, {
        text
