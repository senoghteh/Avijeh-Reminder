import sys
import os
import json
import threading
import time
import traceback
from datetime import datetime, timedelta

import jdatetime
from PyQt5.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout, QPushButton,
    QLabel, QLineEdit, QMessageBox, QCheckBox, QComboBox,
    QScrollArea, QFrame, QSizePolicy, QSpinBox, QDialog,
    QStylePainter, QStyleOptionComboBox, QStyle
)
from PyQt5.QtCore import (
    Qt, QTimer, QPropertyAnimation, QPoint, pyqtSignal, QEvent,
    QSizeF, QRectF
)
from PyQt5.QtGui import (
    QFont, QIcon, QColor, QPainter, QFontMetrics, QTextDocument,
    QPageSize, QTextOption, QTextCursor, QTextTable,
    QTextBlockFormat, QTextTableFormat, QFontDatabase, QTextCharFormat,
    QPageLayout
)
from PyQt5.QtNetwork import QLocalServer, QLocalSocket
from PyQt5.QtPrintSupport import QPrinter, QPrintDialog, QPrintPreviewDialog

try:
    from pystray import Icon as TrayIcon, Menu as TrayMenu, MenuItem as TrayMenuItem
    from PIL import Image, ImageDraw
    TRAY_AVAILABLE = True
except Exception:
    TRAY_AVAILABLE = False

try:
    import winsound
except ImportError:
    winsound = None


APP_VERSION = "1.1.0"
APP_NAME = "AvijehReminder"
APP_TITLE = f"آویژه (نسخه {APP_VERSION})"
APP_DATA_FOLDER = "AvijehReminder"

SINGLE_INSTANCE_KEY = f"{APP_NAME}_single_instance_v1"

DEFAULT_REMIND_INTERVAL = 5
DEFAULT_MAX_REMINDS = 3

MAX_YEAR = 1415


def get_app_data_dir():
    path = os.path.join(
        os.environ.get("APPDATA", os.path.expanduser("~")),
        APP_DATA_FOLDER
    )
    os.makedirs(path, exist_ok=True)
    return path


APP_DATA_DIR = get_app_data_dir()
LOG_FILE = os.path.join(APP_DATA_DIR, "error.log")
TASKS_FILE = os.path.join(APP_DATA_DIR, "tasks.json")

VAZIR_FONT_FAMILY = None


def log_error(msg):
    try:
        with open(LOG_FILE, "a", encoding="utf-8") as f:
            f.write(f"\n[{datetime.now().isoformat()}] ERROR: {msg}\n")
    except Exception:
        pass


def log_info(msg):
    try:
        with open(LOG_FILE, "a", encoding="utf-8") as f:
            f.write(f"\n[{datetime.now().isoformat()}] INFO: {msg}\n")
    except Exception:
        pass


def load_vazir_font():
    global VAZIR_FONT_FAMILY

    candidates = []
    if getattr(sys, "frozen", False):
        base = os.path.dirname(sys.executable)
    else:
        base = os.path.dirname(os.path.abspath(__file__))
    candidates.append(os.path.join(base, "fonts"))
    candidates.append(base)
    candidates.append(os.path.join(APP_DATA_DIR, "fonts"))
    candidates.append(APP_DATA_DIR)

    font_file_names = [
        "Vazir.ttf", "Vazir-Regular.ttf",
        "Vazirmatn-Regular.ttf", "Vazirmatn.ttf", "vazir.ttf",
    ]

    for folder in candidates:
        if not os.path.isdir(folder):
            continue
        for fname in font_file_names:
            fpath = os.path.join(folder, fname)
            if os.path.exists(fpath):
                try:
                    fid = QFontDatabase.addApplicationFont(fpath)
                    if fid != -1:
                        families = QFontDatabase.applicationFontFamilies(fid)
                        if families:
                            VAZIR_FONT_FAMILY = families[0]
                            log_info(f"فونت وزیر بارگذاری شد: {VAZIR_FONT_FAMILY} از {fpath}")
                            return VAZIR_FONT_FAMILY
                except Exception as e:
                    log_error(f"خطا در بارگذاری فونت {fpath}: {e}")

    log_info("فونت وزیر پیدا نشد؛ از Tahoma استفاده می‌شود")
    return None


def show_message_box(parent, icon, title, text,
                     buttons=QMessageBox.Ok,
                     default_button=QMessageBox.Ok):
    msg = QMessageBox(parent)
    msg.setIcon(icon)
    msg.setWindowTitle(title)
    msg.setText(text)
    msg.setStandardButtons(buttons)
    if default_button is not None:
        msg.setDefaultButton(default_button)
    msg.setLayoutDirection(Qt.RightToLeft)

    persian_texts = {
        QMessageBox.Ok: "فهمیدم",
        QMessageBox.Yes: "بله",
        QMessageBox.No: "خیر",
        QMessageBox.Cancel: "انصراف",
        QMessageBox.Close: "بستن",
    }
    for btn_enum, btn_text in persian_texts.items():
        btn = msg.button(btn_enum)
        if btn is not None:
            btn.setText(btn_text)

    return msg.exec_()


def excepthook(exc_type, exc_value, exc_tb):
    log_error("".join(traceback.format_exception(exc_type, exc_value, exc_tb)))


sys.excepthook = excepthook
log_info(f"=== شروع برنامه نسخه {APP_VERSION} ===")


def play_notification_sound():
    if winsound is not None:
        try:
            winsound.PlaySound("SystemAsterisk",
                               winsound.SND_ALIAS | winsound.SND_ASYNC)
            return
        except Exception:
            pass
    try:
        QApplication.beep()
    except Exception:
        pass


JALALI_MONTHS = ["فروردین", "اردیبهشت", "خرداد", "تیر", "مرداد", "شهریور",
                 "مهر", "آبان", "آذر", "دی", "بهمن", "اسفند"]
JALALI_WEEKDAYS = ["شنبه", "یک‌شنبه", "دوشنبه", "سه‌شنبه", "چهارشنبه", "پنج‌شنبه", "جمعه"]


def to_persian_digits(text):
    """تبدیل اعداد لاتین به اعداد فارسی"""
    translation = str.maketrans("0123456789", "۰۱۲۳۴۵۶۷۸۹")
    return str(text).translate(translation)


def days_in_jalali_month(year, month):
    if month <= 6:
        return 31
    if month <= 11:
        return 30
    try:
        jdatetime.date(year, 12, 30)
        return 30
    except ValueError:
        return 29


def today_jalali_str():
    now = jdatetime.datetime.now()
    return f"{now.year}/{now.month:02d}/{now.day:02d}"


def is_past_jalali_datetime(year, month, day, hour, minute):
    selected = jdatetime.datetime(year, month, day, hour, minute)
    return selected < jdatetime.datetime.now()


def get_executable_path():
    if getattr(sys, "frozen", False):
        return sys.executable
    python_dir = os.path.dirname(sys.executable)
    pythonw = os.path.join(python_dir, "pythonw.exe")
    if os.path.exists(pythonw):
        return f'"{pythonw}" "{os.path.abspath(__file__)}"'
    return f'"{sys.executable}" "{os.path.abspath(__file__)}"'


def is_startup_enabled():
    if os.name != "nt":
        return False
    import winreg
    try:
        key = winreg.OpenKey(
            winreg.HKEY_CURRENT_USER,
            r"Software\Microsoft\Windows\CurrentVersion\Run",
            0, winreg.KEY_READ
        )
        try:
            winreg.QueryValueEx(key, APP_NAME)
            return True
        except FileNotFoundError:
            return False
        finally:
            winreg.CloseKey(key)
    except Exception:
        return False


def set_startup(enabled: bool):
    if os.name != "nt":
        return
    import winreg
    try:
        key = winreg.OpenKey(
            winreg.HKEY_CURRENT_USER,
            r"Software\Microsoft\Windows\CurrentVersion\Run",
            0, winreg.KEY_SET_VALUE
        )
        try:
            if enabled:
                winreg.SetValueEx(key, APP_NAME, 0, winreg.REG_SZ, get_executable_path())
            else:
                try:
                    winreg.DeleteValue(key, APP_NAME)
                except FileNotFoundError:
                    pass
        finally:
            winreg.CloseKey(key)
    except Exception as e:
        log_error(f"خطا در تنظیم Startup: {e}")


def activate_existing_instance(timeout_ms=800):
    socket = QLocalSocket()
    socket.connectToServer(SINGLE_INSTANCE_KEY)
    if socket.waitForConnected(timeout_ms):
        try:
            socket.write(b"SHOW")
            socket.waitForBytesWritten(timeout_ms)
        except Exception as e:
            log_error(f"خطا در ارسال پیام به نمونه موجود: {e}")
        try:
            socket.disconnectFromServer()
        except Exception:
            pass
        return True
    return False


def load_tasks():
    if os.path.exists(TASKS_FILE):
        try:
            with open(TASKS_FILE, "r", encoding="utf-8") as f:
                tasks = json.load(f)
            today = today_jalali_str()
            for t in tasks:
                if "date" not in t or not t["date"]:
                    t["date"] = today
                if "last_triggered" not in t:
                    t["last_triggered"] = None
                if "snooze_until" not in t:
                    t["snooze_until"] = None
                if "done" not in t:
                    t["done"] = False
                if "forgotten" not in t:
                    t["forgotten"] = False
                if "remind_interval" not in t:
                    t["remind_interval"] = DEFAULT_REMIND_INTERVAL
                if "max_reminds" not in t:
                    t["max_reminds"] = DEFAULT_MAX_REMINDS
                if "remind_count" not in t:
                    t["remind_count"] = 0
            return tasks
        except Exception as e:
            log_error(f"خطا در بارگذاری تسک‌ها: {e}")
            return []
    return []


def save_tasks(tasks):
    try:
        with open(TASKS_FILE, "w", encoding="utf-8") as f:
            json.dump(tasks, f, ensure_ascii=False, indent=2)
    except Exception as e:
        log_error(f"خطا در ذخیره تسک‌ها: {e}")


def get_jalali_date_string():
    now = jdatetime.datetime.now()
    return f"{JALALI_WEEKDAYS[now.weekday()]} {now.day} {JALALI_MONTHS[now.month - 1]} {now.year}"


def format_jalali_date_short(date_str):
    try:
        y, m, d = map(int, date_str.split("/"))
        name = JALALI_MONTHS[m - 1]
        return f"{d} {name}"
    except Exception:
        return date_str


def create_default_icon_image(size=64):
    img = Image.new("RGBA", (size, size), (0, 0, 0, 0))
    draw = ImageDraw.Draw(img)
    margin = size // 8
    draw.ellipse([margin, margin, size - margin, size - margin], fill="#0078d4")
    inner = size // 3
    draw.ellipse([inner, inner, size - inner, size - inner], fill="white")
    return img


def ensure_icon_file():
    ico_path = os.path.join(APP_DATA_DIR, "app.ico")
    if not os.path.exists(ico_path):
        try:
            img = create_default_icon_image(256)
            img.save(ico_path, format="ICO",
                     sizes=[(16, 16), (32, 32), (48, 48), (64, 64), (128, 128), (256, 256)])
        except Exception as e:
            log_error(f"خطا در ساخت آیکون: {e}")
    return ico_path


def _style_combo_center(combo):
    for i in range(combo.count()):
        combo.setItemData(i, Qt.AlignCenter, Qt.TextAlignmentRole)


class CenterComboBox(QComboBox):
    def paintEvent(self, event):
        try:
            painter = QStylePainter(self)
            opt = QStyleOptionComboBox()
            self.initStyleOption(opt)
            painter.drawComplexControl(QStyle.CC_ComboBox, opt)
            text_rect = self.style().subControlRect(
                QStyle.CC_ComboBox, opt, QStyle.SC_ComboBoxEditField, self
            )
            painter.drawText(text_rect, Qt.AlignCenter, opt.currentText)
        except Exception:
            super().paintEvent(event)


class RTLPlaceholderLineEdit(QLineEdit):
    def __init__(self, placeholder_text="", parent=None):
        super().__init__(parent)
        self.setLayoutDirection(Qt.RightToLeft)
        self.setAlignment(Qt.AlignLeft | Qt.AlignVCenter)
        self.setPlaceholderText("\u200F" + placeholder_text)

    def focusInEvent(self, event):
        super().focusInEvent(event)
        if not self.text():
            QTimer.singleShot(0, self._move_cursor_right)

    def mousePressEvent(self, event):
        super().mousePressEvent(event)
        if not self.text():
            QTimer.singleShot(0, self._move_cursor_right)

    def _move_cursor_right(self):
        try:
            self.setCursorPosition(0)
        except Exception:
            pass


# ---------- دیالوگ ویرایش کار ----------
class EditTaskDialog(QDialog):
    def __init__(self, task, parent=None):
        super().__init__(parent)
        self.setWindowTitle("ویرایش کار")
        self.setLayoutDirection(Qt.LeftToRight)
        self.setModal(True)
        self.setFixedWidth(560)

        self.new_title = task.get("title", "")
        self.new_date = task.get("date", today_jalali_str())
        self.new_time = task.get("time", "00:00")
        self.new_remind_interval = task.get("remind_interval", DEFAULT_REMIND_INTERVAL)
        self.new_max_reminds = task.get("max_reminds", DEFAULT_MAX_REMINDS)

        layout = QVBoxLayout(self)
        layout.setSpacing(10)
        layout.setContentsMargins(16, 16, 16, 16)

        title_row = QHBoxLayout()
        title_row.setSpacing(8)

        self.title_input = RTLPlaceholderLineEdit("عنوان کار را وارد کنید...")
        self.title_input.setText(task.get("title", ""))
        self.title_input.setFont(QFont("Tahoma", 11))
        self.title_input.setMinimumHeight(30)
        self.title_input.setStyleSheet("""
            QLineEdit {
                padding: 4px 8px;
                border: 1px solid #ccc;
                border-radius: 5px;
                background: #ffffff;
                color: #000000;
            }
            QLineEdit:focus { border: 1px solid #0078d4; }
        """)
        self.title_input.returnPressed.connect(self._on_save)
        title_row.addWidget(self.title_input, 1)

        lbl_title = QLabel("عنوان:")
        lbl_title.setFont(QFont("Tahoma", 10))
        lbl_title.setFixedWidth(50)
        lbl_title.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
        title_row.addWidget(lbl_title)

        layout.addLayout(title_row)

        dt_row = QHBoxLayout()
        dt_row.setSpacing(4)

        try:
            y, m, d = map(int, task.get("date", today_jalali_str()).split("/"))
        except Exception:
            now = jdatetime.datetime.now()
            y, m, d = now.year, now.month, now.day
        try:
            hh, mm = map(int, task.get("time", "00:00").split(":"))
        except Exception:
            hh, mm = 0, 0

        self.hour_spin = QSpinBox()
        self.hour_spin.setRange(0, 23)
        self.hour_spin.setValue(hh)
        self.hour_spin.setFixedWidth(50)
        self.hour_spin.setFixedHeight(26)
        self.hour_spin.setFont(QFont("Tahoma", 10))
        self.hour_spin.setAlignment(Qt.AlignCenter)
        self.hour_spin.setLayoutDirection(Qt.LeftToRight)

        self.minute_spin = QSpinBox()
        self.minute_spin.setRange(0, 59)
        self.minute_spin.setValue(mm)
        self.minute_spin.setFixedWidth(50)
        self.minute_spin.setFixedHeight(26)
        self.minute_spin.setFont(QFont("Tahoma", 10))
        self.minute_spin.setAlignment(Qt.AlignCenter)
        self.minute_spin.setLayoutDirection(Qt.LeftToRight)

        self.year_combo = CenterComboBox()
        self.year_combo.setFont(QFont("Tahoma", 10))
        start_year = max(1300, y - 2)
        for yy in range(start_year, MAX_YEAR + 1):
            self.year_combo.addItem(str(yy))
        self.year_combo.setCurrentText(str(y))
        self.year_combo.setFixedWidth(64)
        self.year_combo.setFixedHeight(26)
        self.year_combo.setLayoutDirection(Qt.LeftToRight)
        _style_combo_center(self.year_combo)

        self.month_combo = CenterComboBox()
        self.month_combo.setFont(QFont("Tahoma", 10))
        for i, name in enumerate(JALALI_MONTHS):
            self.month_combo.addItem(name, i + 1)
        self.month_combo.setCurrentIndex(m - 1)
        self.month_combo.setFixedWidth(110)
        self.month_combo.setFixedHeight(26)
        self.month_combo.setLayoutDirection(Qt.LeftToRight)
        _style_combo_center(self.month_combo)

        self.day_combo = CenterComboBox()
        self.day_combo.setFont(QFont("Tahoma", 10))
        self.day_combo.setFixedWidth(52)
        self.day_combo.setFixedHeight(26)
        self.day_combo.setLayoutDirection(Qt.LeftToRight)
        self._populate_days(y, m, d)

        self.month_combo.currentIndexChanged.connect(self._update_days)
        self.year_combo.currentIndexChanged.connect(self._update_days)

        dt_row.addStretch()
        dt_row.addWidget(self.hour_spin)
        dt_row.addWidget(QLabel(":"))
        dt_row.addWidget(self.minute_spin)
        lbl_time = QLabel("ساعت:")
        lbl_time.setFont(QFont("Tahoma", 10))
        dt_row.addWidget(lbl_time)

        dt_row.addSpacing(10)

        dt_row.addWidget(self.year_combo)
        dt_row.addWidget(self.month_combo)
        dt_row.addWidget(self.day_combo)
        lbl_date = QLabel("تاریخ:")
        lbl_date.setFont(QFont("Tahoma", 10))
        dt_row.addWidget(lbl_date)

        layout.addLayout(dt_row)

        remind_row = QHBoxLayout()
        remind_row.setSpacing(6)

        self.remind_interval_spin = QSpinBox()
        self.remind_interval_spin.setRange(1, 120)
        self.remind_interval_spin.setValue(task.get("remind_interval", DEFAULT_REMIND_INTERVAL))
        self.remind_interval_spin.setSuffix(" دقیقه")
        self.remind_interval_spin.setFixedWidth(100)
        self.remind_interval_spin.setFixedHeight(26)
        self.remind_interval_spin.setFont(QFont("Tahoma", 10))
        self.remind_interval_spin.setLayoutDirection(Qt.LeftToRight)
        self.remind_interval_spin.setToolTip("فاصله زمانی بین یادآوری‌ها")

        self.max_reminds_spin = QSpinBox()
        self.max_reminds_spin.setRange(1, 20)
        self.max_reminds_spin.setValue(task.get("max_reminds", DEFAULT_MAX_REMINDS))
        self.max_reminds_spin.setSuffix(" بار")
        self.max_reminds_spin.setFixedWidth(84)
        self.max_reminds_spin.setFixedHeight(26)
        self.max_reminds_spin.setFont(QFont("Tahoma", 10))
        self.max_reminds_spin.setLayoutDirection(Qt.LeftToRight)
        self.max_reminds_spin.setToolTip("حداکثر تعداد تکرار یادآوری")

        remind_row.addStretch()
        remind_row.addWidget(self.remind_interval_spin)
        lbl_ri = QLabel("فاصله یادآوری:")
        lbl_ri.setFont(QFont("Tahoma", 10))
        remind_row.addWidget(lbl_ri)
        remind_row.addSpacing(10)
        remind_row.addWidget(self.max_reminds_spin)
        lbl_mr = QLabel("حداکثر تکرار:")
        lbl_mr.setFont(QFont("Tahoma", 10))
        remind_row.addWidget(lbl_mr)

        layout.addLayout(remind_row)
        layout.addStretch()

        btn_row = QHBoxLayout()
        btn_row.addStretch()

        save_btn = QPushButton("ذخیره")
        save_btn.setFixedWidth(90)
        save_btn.setMinimumHeight(32)
        save_btn.setFont(QFont("Tahoma", 10))
        save_btn.setCursor(Qt.PointingHandCursor)
        save_btn.setStyleSheet("""
            QPushButton { background-color: #0078d4; color: white; border-radius: 5px; padding: 4px; }
            QPushButton:hover { background-color: #1a8ae0; }
        """)
        save_btn.clicked.connect(self._on_save)

        cancel_btn = QPushButton("انصراف")
        cancel_btn.setFixedWidth(90)
        cancel_btn.setMinimumHeight(32)
        cancel_btn.setFont(QFont("Tahoma", 10))
        cancel_btn.setCursor(Qt.PointingHandCursor)
        cancel_btn.setStyleSheet("""
            QPushButton { background-color: #e0e0e0; color: #333; border-radius: 5px; padding: 4px; }
            QPushButton:hover { background-color: #d0d0d0; }
        """)
        cancel_btn.clicked.connect(self.reject)

        btn_row.addWidget(save_btn)
        btn_row.addWidget(cancel_btn)
        layout.addLayout(btn_row)

        QTimer.singleShot(0, self.title_input.setFocus)

    def _populate_days(self, year, month, current_day):
        max_days = days_in_jalali_month(year, month)
        self.day_combo.blockSignals(True)
        self.day_combo.clear()
        for d in range(1, max_days + 1):
            self.day_combo.addItem(str(d))
        _style_combo_center(self.day_combo)
        if 1 <= current_day <= max_days:
            self.day_combo.setCurrentText(str(current_day))
        self.day_combo.blockSignals(False)

    def _update_days(self):
        try:
            y = int(self.year_combo.currentText())
            m = self.month_combo.currentIndex() + 1
            cur = self.day_combo.currentText()
            cur_d = int(cur) if cur.isdigit() else 1
            self._populate_days(y, m, cur_d)
        except Exception:
            pass

    def _on_save(self):
        title = self.title_input.text().strip()
        if not title:
            show_message_box(self, QMessageBox.Warning, "خطا", "عنوان کار را وارد کنید.")
            return
        try:
            y = int(self.year_combo.currentText())
            m = self.month_combo.currentIndex() + 1
            d = int(self.day_combo.currentText())
        except Exception:
            show_message_box(self, QMessageBox.Warning, "خطا", "تاریخ معتبر نیست.")
            return

        hour = self.hour_spin.value()
        minute = self.minute_spin.value()

        try:
            if is_past_jalali_datetime(y, m, d, hour, minute):
                show_message_box(
                    self, QMessageBox.Warning, "خطا",
                    "گذشته‌ها گذشته به فکر آینده باش"
                )
                return
        except Exception:
            show_message_box(self, QMessageBox.Warning, "خطا", "تاریخ و ساعت معتبر نیست.")
            return

        self.new_title = title
        self.new_date = f"{y}/{m:02d}/{d:02d}"
        self.new_time = f"{hour:02d}:{minute:02d}"
        self.new_remind_interval = self.remind_interval_spin.value()
        self.new_max_reminds = self.max_reminds_spin.value()
        self.accept()


# ---------- ویجت اعلان کارهای امروز ----------
class TodayTasksNotification(QWidget):
    closed = pyqtSignal()

    def __init__(self, tasks, parent=None):
        super().__init__(parent)
        self.setWindowFlags(
            Qt.Window | Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint
        )
        self.setAttribute(Qt.WA_ShowWithoutActivating, True)

        n = len(tasks)
        width = 420
        line_h = 30
        list_h = min(n * line_h + 12, 260)
        height = 55 + list_h + 60 + 20
        self.setFixedSize(width, height)

        self.setStyleSheet("""
            QWidget#root {
                background-color: #2b2b2b;
                border: 2px solid #0078d4;
                border-radius: 10px;
            }
            QPushButton#okBtn {
                background-color: #0078d4;
                color: #ffffff;
                border: none;
                border-radius: 6px;
                padding: 8px 16px;
                font-family: Tahoma;
                font-size: 12px;
                font-weight: bold;
            }
            QPushButton#okBtn:hover { background-color: #1a8ae0; }
        """)

        outer = QVBoxLayout(self)
        outer.setContentsMargins(0, 0, 0, 0)
        root = QWidget()
        root.setObjectName("root")
        outer.addWidget(root)

        main = QVBoxLayout(root)
        main.setContentsMargins(14, 12, 14, 12)
        main.setSpacing(6)

        header_row = QHBoxLayout()
        header_row.setContentsMargins(0, 0, 0, 0)
        header_row.addStretch()
        header = QLabel(f"📋  کارهای امروز  ({n} مورد)")
        header.setStyleSheet(
            "color: #4fc3f7; background-color: transparent; "
            "font-family: Tahoma; font-size: 13px; font-weight: bold; "
            "border: none;"
        )
        header.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
        header.setLayoutDirection(Qt.RightToLeft)
        header_row.addWidget(header)
        main.addLayout(header_row)

        scroll = QScrollArea()
        scroll.setWidgetResizable(True)
        scroll.setHorizontalScrollBarPolicy(Qt.ScrollBarAlwaysOff)
        scroll.setFrameShape(QFrame.NoFrame)
        scroll.setStyleSheet("""
            QScrollArea { background: transparent; border: none; }
            QScrollBar:vertical {
                background: transparent;
                width: 6px;
                margin: 0;
            }
            QScrollBar::handle:vertical {
                background: #555;
                border-radius: 3px;
                min-height: 20px;
            }
        """)
        scroll.viewport().setStyleSheet("background-color: transparent;")
        scroll.viewport().setAutoFillBackground(False)

        list_widget = QWidget()
        list_widget.setStyleSheet("background-color: transparent;")
        list_widget.setAutoFillBackground(False)
        list_widget.setLayoutDirection(Qt.LeftToRight)

        list_layout = QVBoxLayout(list_widget)
        list_layout.setContentsMargins(0, 0, 0, 0)
        list_layout.setSpacing(4)

        icon_w = 26
        label_padding_h = 16
        available_text_w = width - 28 - 6 - icon_w - label_padding_h - 4

        for t in tasks:
            row = QFrame()
            row.setStyleSheet("QFrame { background: transparent; border: none; }")
            row_layout = QHBoxLayout(row)
            row_layout.setContentsMargins(0, 0, 0, 0)
            row_layout.setSpacing(0)

            row_layout.addStretch()

            text = f"{t['time']}  |  {t['title']}"
            lbl = QLabel()
            lbl.setStyleSheet(
                "color: #ffffff; background-color: transparent; "
                "font-family: Tahoma; padding: 4px 8px; border: none;"
            )
            lbl.setWordWrap(False)
            lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
            lbl.setLayoutDirection(Qt.RightToLeft)
            lbl.setSizePolicy(QSizePolicy.Maximum, QSizePolicy.Preferred)

            base_size = 11
            min_size = 7
            font = QFont("Tahoma", base_size)
            fm = QFontMetrics(font)
            while (font.pointSize() > min_size
                   and fm.horizontalAdvance(text) > available_text_w):
                font.setPointSize(font.pointSize() - 1)
                fm = QFontMetrics(font)

            final_text = text
            if fm.horizontalAdvance(text) > available_text_w:
                final_text = fm.elidedText(text, Qt.ElideRight, available_text_w)

            lbl.setText(final_text)
            lbl.setFont(font)
            row_layout.addWidget(lbl, 0, Qt.AlignRight | Qt.AlignVCenter)

            icon_lbl = QLabel("⏰")
            icon_lbl.setFont(QFont("Tahoma", 12))
            icon_lbl.setFixedWidth(icon_w)
            icon_lbl.setAlignment(Qt.AlignCenter)
            icon_lbl.setStyleSheet(
                "color: #ffffff; background-color: transparent; border: none;"
            )
            row_layout.addWidget(icon_lbl, 0, Qt.AlignRight | Qt.AlignVCenter)

            list_layout.addWidget(row)

        list_layout.addStretch()
        scroll.setWidget(list_widget)
        scroll.setFixedHeight(list_h)
        main.addWidget(scroll)

        btn_row = QHBoxLayout()
        btn_row.addStretch()
        ok_btn = QPushButton("متوجه شدم")
        ok_btn.setObjectName("okBtn")
        ok_btn.setFixedWidth(130)
        ok_btn.setMinimumHeight(34)
        ok_btn.setCursor(Qt.PointingHandCursor)
        ok_btn.clicked.connect(self._dismiss)
        btn_row.addWidget(ok_btn)
        btn_row.addStretch()
        main.addLayout(btn_row)

        self._position_center()

    def _position_center(self):
        try:
            screen = QApplication.primaryScreen()
            if screen is not None:
                geo = screen.availableGeometry()
                x = geo.center().x() - self.width() // 2
                y = geo.center().y() - self.height() // 2
                self.move(x, y)
            else:
                self.move(300, 200)
        except Exception as e:
            log_error(f"خطا در موقعیت اعلان امروز: {e}")
            self.move(300, 200)

    def show_notification(self):
        self._position_center()
        self.show()
        self.raise_()
        try:
            self.activateWindow()
        except Exception:
            pass

    def _dismiss(self):
        try:
            self.closed.emit()
        except Exception:
            pass
        self.close()


# ---------- ویجت اعلان کار (زمان مقرر) ----------
class NotificationWindow(QWidget):
    closed = pyqtSignal()

    def __init__(self, title, time_str, task_id,
                 remind_count=0, max_reminds=DEFAULT_MAX_REMINDS,
                 remind_interval=DEFAULT_REMIND_INTERVAL, parent=None):
        super().__init__(parent)
        self.task_id = task_id
        self.remind_count = remind_count
        self.max_reminds = max_reminds
        self.remind_interval = remind_interval

        self.setWindowFlags(
            Qt.Window | Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint
        )
        self.setAttribute(Qt.WA_ShowWithoutActivating, True)
        self.setFixedSize(400, 200)

        self.setStyleSheet("""
            QWidget#root {
                background-color: #2b2b2b;
                border: 2px solid #0078d4;
                border-radius: 10px;
            }
            QPushButton#doneBtn {
                background-color: #28a745;
                color: #ffffff;
                border: none;
                border-radius: 6px;
                padding: 6px 12px;
                font-family: Tahoma;
                font-size: 11px;
                font-weight: bold;
            }
            QPushButton#doneBtn:hover { background-color: #218838; }
            QPushButton#remindBtn {
                background-color: #0078d4;
                color: #ffffff;
                border: none;
                border-radius: 6px;
                padding: 6px 12px;
                font-family: Tahoma;
                font-size: 11px;
                font-weight: bold;
            }
            QPushButton#remindBtn:hover { background-color: #1a8ae0; }
            QPushButton#endRemindBtn {
                background-color: #f0ad4e;
                color: #ffffff;
                border: none;
                border-radius: 6px;
                padding: 6px 12px;
                font-family: Tahoma;
                font-size: 11px;
                font-weight: bold;
            }
            QPushButton#endRemindBtn:hover { background-color: #ec971f; }
        """)

        outer = QVBoxLayout(self)
        outer.setContentsMargins(0, 0, 0, 0)
        root = QWidget()
        root.setObjectName("root")
        outer.addWidget(root)

        main = QVBoxLayout(root)
        main.setContentsMargins(12, 10, 12, 10)
        main.setSpacing(6)

        header_row = QHBoxLayout()
        header_row.setContentsMargins(0, 0, 0, 0)
        header_row.addStretch()
        header = QLabel("یادآور آویژه")
        header.setStyleSheet(
            "color: #4fc3f7; background-color: transparent; "
            "font-family: Tahoma; font-size: 9pt; border: none;"
        )
        header.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
        header.setLayoutDirection(Qt.RightToLeft)
        header_row.addWidget(header)
        main.addLayout(header_row)

        title_row = QHBoxLayout()
        title_row.setContentsMargins(0, 0, 0, 0)
        title_row.setSpacing(4)
        title_row.addStretch(1)

        display_title = title if title else "—"
        title_lbl = QLabel(display_title)
        title_lbl.setStyleSheet(
            "color: #ffffff; background-color: transparent; border: none;"
        )
        title_lbl.setWordWrap(False)
        title_lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
        title_lbl.setLayoutDirection(Qt.RightToLeft)
        title_lbl.setSizePolicy(QSizePolicy.Maximum, QSizePolicy.Preferred)

        icon_w = 28
        available_width = max(self.width() - 24 - 6 - icon_w, 100)
        base_size = 13
        min_size = 7
        font = QFont("Tahoma", base_size, QFont.Bold)
        fm = QFontMetrics(font)
        while font.pointSize() > min_size and fm.horizontalAdvance(display_title) > available_width:
            font.setPointSize(font.pointSize() - 1)
            fm = QFontMetrics(font)

        if fm.horizontalAdvance(display_title) > available_width:
            display_title = fm.elidedText(display_title, Qt.ElideRight, available_width)
            title_lbl.setText(display_title)

        title_lbl.setFont(font)

        title_row.addWidget(title_lbl, 0, Qt.AlignRight | Qt.AlignVCenter)

        icon_lbl = QLabel("⏰")
        icon_lbl.setFont(QFont("Tahoma", 14, QFont.Bold))
        icon_lbl.setFixedWidth(icon_w)
        icon_lbl.setAlignment(Qt.AlignCenter)
        icon_lbl.setStyleSheet(
            "color: #ffffff; background-color: transparent; border: none;"
        )
        title_row.addWidget(icon_lbl, 0, Qt.AlignRight | Qt.AlignVCenter)
        main.addLayout(title_row)

        time_row = QHBoxLayout()
        time_row.setContentsMargins(0, 0, 0, 0)
        time_row.addStretch()
        time_lbl = QLabel(f"🕐  ساعت مقرر: {time_str}")
        time_lbl.setStyleSheet(
            "color: #b0e0ff; background-color: transparent; "
            "font-family: Tahoma; font-size: 11pt; border: none;"
        )
        time_lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
        time_lbl.setLayoutDirection(Qt.RightToLeft)
        time_row.addWidget(time_lbl)
        main.addLayout(time_row)

        info_row = QHBoxLayout()
        info_row.setContentsMargins(0, 0, 0, 0)
        info_row.addStretch()

        is_last = (self.remind_count >= self.max_reminds)
        if is_last:
            info_text = "آخرین یادآوری — مواظب باش که انجام این کار را فراموش نکنی"
            info_color = "#f0ad4e"
        else:
            remaining = self.max_reminds - self.remind_count
            info_text = (f"یادآوری بعدی پس از {self.remind_interval} دقیقه "
                         f"(باقی‌مانده: {remaining} بار)")
            info_color = "#aaaaaa"

        info_lbl = QLabel(info_text)
        info_lbl.setStyleSheet(
            f"color: {info_color}; background-color: transparent; "
            "font-family: Tahoma; font-size: 8pt; border: none;"
        )
        info_lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
        info_lbl.setLayoutDirection(Qt.RightToLeft)
        info_row.addWidget(info_lbl)
        main.addLayout(info_row)

        main.addStretch()

        btns = QHBoxLayout()
        btns.setSpacing(6)

        done_btn = QPushButton("انجام دادم")
        done_btn.setObjectName("doneBtn")
        done_btn.setMinimumHeight(32)
        done_btn.setCursor(Qt.PointingHandCursor)
        done_btn.clicked.connect(self._mark_done)

        if is_last:
            remind_btn_text = "پایان یادآوری"
            remind_btn_style = "endRemindBtn"
        else:
            remind_btn_text = "یادآوری کن"
            remind_btn_style = "remindBtn"

        remind_btn = QPushButton(remind_btn_text)
        remind_btn.setObjectName(remind_btn_style)
        remind_btn.setMinimumHeight(32)
        remind_btn.setCursor(Qt.PointingHandCursor)
        remind_btn.clicked.connect(self._on_remind_clicked)

        btns.addWidget(done_btn)
        btns.addWidget(remind_btn)
        main.addLayout(btns)

        self._position_bottom_right()

    def _position_bottom_right(self):
        try:
            screen = QApplication.primaryScreen()
            if screen is not None:
                geo = screen.availableGeometry()
                x = geo.right() - self.width() - 20
                y = geo.bottom() - self.height() - 20
                self.move(x, y)
            else:
                self.move(100, 100)
        except Exception as e:
            log_error(f"خطا در موقعیت اعلان: {e}")
            self.move(100, 100)

    def show_notification(self):
        self._position_bottom_right()
        self.show()
        self.raise_()
        try:
            self.activateWindow()
        except Exception:
            pass

    def _mark_done(self):
        log_info(f"_mark_done called: task_id={self.task_id}")
        try:
            tasks = load_tasks()
            for t in tasks:
                if t["id"] == self.task_id:
                    t["done"] = True
                    t["forgotten"] = False
                    t["snooze_until"] = None
                    t["remind_count"] = 0
                    break
            save_tasks(tasks)
            self._refresh_main_window()
        except Exception as e:
            log_error(f"خطا در علامت‌گذاری انجام شد: {e}")
        self._close()

    def _on_remind_clicked(self):
        log_info(
            f"_on_remind_clicked: task_id={self.task_id}, "
            f"count={self.remind_count}, max={self.max_reminds}"
        )
        try:
            tasks = load_tasks()
            target = None
            for t in tasks:
                if t["id"] == self.task_id:
                    target = t
                    break

            if target is None:
                log_info(f"task_id={self.task_id} یافت نشد")
                self._close()
                return

            if self.remind_count >= self.max_reminds:
                target["forgotten"] = True
                target["snooze_until"] = None
                target["remind_count"] = self.remind_count
                log_info(f"Task {self.task_id} → فراموش شد ")
            else:
                target["remind_count"] = self.remind_count + 1
                next_time = datetime.now() + timedelta(minutes=self.remind_interval)
                target["snooze_until"] = next_time.isoformat()
                log_info(
                    f"Task {self.task_id} → snooze تا {next_time.isoformat()} "
                    f"(یادآوری {target['remind_count']} از {self.max_reminds})"
                )
            save_tasks(tasks)
            self._refresh_main_window()
        except Exception as e:
            log_error(f"خطا در _on_remind_clicked: {e}\n{traceback.format_exc()}")
        self._close()

    def _refresh_main_window(self):
        try:
            app = QApplication.instance()
            if app is not None and hasattr(app, "_reminder_window"):
                w = app._reminder_window
                w.tasks = load_tasks()
                QTimer.singleShot(0, w._refresh_task_list)
        except Exception as e:
            log_error(f"خطا در به‌روزرسانی پنجره اصلی: {e}")

    def _close(self):
        try:
            self.closed.emit()
        except Exception:
            pass
        self.close()


# ---------- پنجره اصلی ----------
class ReminderApp(QWidget):
    def __init__(self):
        super().__init__()
        self.tasks = load_tasks()
        self._active_notifications = []
        self._selected_task_id = None
        self._init_ui()
        self._setup_tray()
        self._setup_single_instance_server()
        self._start_scheduler()
        self._refresh_task_list()

        QTimer.singleShot(1500, self._show_today_tasks_notification)

    def _init_ui(self):
        self.setWindowTitle(APP_TITLE)
        self.setFixedSize(600, 560)
        self.setLayoutDirection(Qt.LeftToRight)

        ico = ensure_icon_file()
        if os.path.exists(ico):
            self.setWindowIcon(QIcon(ico))

        layout = QVBoxLayout()
        layout.setSpacing(5)
        layout.setContentsMargins(10, 8, 10, 6)

        self.date_label = QLabel(get_jalali_date_string())
        self.date_label.setFont(QFont("Tahoma", 11, QFont.Bold))
        self.date_label.setAlignment(Qt.AlignCenter)
        self.date_label.setStyleSheet("color: #0078d4; padding: 2px;")
        layout.addWidget(self.date_label)

        self.task_input = RTLPlaceholderLineEdit("عنوان کار را وارد کنید...")
        self.task_input.setFont(QFont("Tahoma", 10))
        self.task_input.setMinimumHeight(28)
        self.task_input.setStyleSheet("""
            QLineEdit {
                padding: 3px 8px;
                border: 1px solid #ccc;
                border-radius: 5px;
                background: #ffffff;
                color: #000000;
            }
            QLineEdit:focus { border: 1px solid #0078d4; }
        """)
        self.task_input.returnPressed.connect(self._add_task)
        layout.addWidget(self.task_input)

        row2 = QHBoxLayout()
        row2.setSpacing(3)

        today = jdatetime.datetime.now()
        now_time = datetime.now()

        row2.addStretch()

        add_btn = QPushButton("افزودن")
        add_btn.setFont(QFont("Tahoma", 9))
        add_btn.setFixedWidth(58)
        add_btn.setMinimumHeight(24)
        add_btn.setStyleSheet("""
            QPushButton { background-color: #0078d4; color: white; border-radius: 5px; padding: 3px; }
            QPushButton:hover { background-color: #1a8ae0; }
        """)
        add_btn.clicked.connect(self._add_task)
        row2.addWidget(add_btn)

        self.hour_spin = QSpinBox()
        self.hour_spin.setRange(0, 23)
        self.hour_spin.setValue(now_time.hour)
        self.hour_spin.setFixedWidth(42)
        self.hour_spin.setFixedHeight(24)
        self.hour_spin.setFont(QFont("Tahoma", 9))
        self.hour_spin.setAlignment(Qt.AlignCenter)
        self.hour_spin.setLayoutDirection(Qt.LeftToRight)
        row2.addWidget(self.hour_spin)

        lbl_colon = QLabel(":")
        lbl_colon.setFont(QFont("Tahoma", 10, QFont.Bold))
        row2.addWidget(lbl_colon)

        self.minute_spin = QSpinBox()
        self.minute_spin.setRange(0, 59)
        self.minute_spin.setValue(now_time.minute)
        self.minute_spin.setFixedWidth(42)
        self.minute_spin.setFixedHeight(24)
        self.minute_spin.setFont(QFont("Tahoma", 9))
        self.minute_spin.setAlignment(Qt.AlignCenter)
        self.minute_spin.setLayoutDirection(Qt.LeftToRight)
        row2.addWidget(self.minute_spin)

        lbl_time = QLabel("ساعت:")
        lbl_time.setFont(QFont("Tahoma", 9))
        row2.addWidget(lbl_time)

        self.year_combo = CenterComboBox()
        self.year_combo.setFont(QFont("Tahoma", 9))
        for y in range(today.year, MAX_YEAR + 1):
            self.year_combo.addItem(str(y))
        self.year_combo.setFixedWidth(58)
        self.year_combo.setFixedHeight(24)
        self.year_combo.setLayoutDirection(Qt.LeftToRight)
        _style_combo_center(self.year_combo)
        row2.addWidget(self.year_combo)

        self.month_combo = CenterComboBox()
        self.month_combo.setFont(QFont("Tahoma", 9))
        for i, m in enumerate(JALALI_MONTHS):
            self.month_combo.addItem(m, i + 1)
        self.month_combo.setCurrentIndex(today.month - 1)
        self.month_combo.setFixedWidth(90)
        self.month_combo.setFixedHeight(24)
        self.month_combo.setLayoutDirection(Qt.LeftToRight)
        _style_combo_center(self.month_combo)
        row2.addWidget(self.month_combo)

        self.day_combo = CenterComboBox()
        self.day_combo.setFont(QFont("Tahoma", 9))
        self.day_combo.setFixedWidth(44)
        self.day_combo.setFixedHeight(24)
        self.day_combo.setLayoutDirection(Qt.LeftToRight)
        row2.addWidget(self.day_combo)

        lbl_date = QLabel("تاریخ:")
        lbl_date.setFont(QFont("Tahoma", 9))
        row2.addWidget(lbl_date)

        self.day_combo.clear()
        max_days = days_in_jalali_month(today.year, today.month)
        for d in range(1, max_days + 1):
            self.day_combo.addItem(str(d))
        _style_combo_center(self.day_combo)
        self.day_combo.setCurrentText(str(today.day))

        self.month_combo.currentIndexChanged.connect(self._update_days)
        self.year_combo.currentIndexChanged.connect(self._update_days)

        layout.addLayout(row2)

        row3 = QHBoxLayout()
        row3.setSpacing(5)
        row3.addStretch()

        self.new_remind_interval_spin = QSpinBox()
        self.new_remind_interval_spin.setRange(1, 120)
        self.new_remind_interval_spin.setValue(DEFAULT_REMIND_INTERVAL)
        self.new_remind_interval_spin.setSuffix(" دقیقه")
        self.new_remind_interval_spin.setFixedWidth(92)
        self.new_remind_interval_spin.setFixedHeight(24)
        self.new_remind_interval_spin.setFont(QFont("Tahoma", 9))
        self.new_remind_interval_spin.setLayoutDirection(Qt.LeftToRight)
        self.new_remind_interval_spin.setToolTip("فاصله بین یادآوری‌ها برای کار جدید")
        row3.addWidget(self.new_remind_interval_spin)

        lbl_ri = QLabel("فاصله یادآوری:")
        lbl_ri.setFont(QFont("Tahoma", 9))
        row3.addWidget(lbl_ri)

        row3.addSpacing(8)

        self.new_max_reminds_spin = QSpinBox()
        self.new_max_reminds_spin.setRange(1, 20)
        self.new_max_reminds_spin.setValue(DEFAULT_MAX_REMINDS)
        self.new_max_reminds_spin.setSuffix(" بار")
        self.new_max_reminds_spin.setFixedWidth(78)
        self.new_max_reminds_spin.setFixedHeight(24)
        self.new_max_reminds_spin.setFont(QFont("Tahoma", 9))
        self.new_max_reminds_spin.setLayoutDirection(Qt.LeftToRight)
        self.new_max_reminds_spin.setToolTip("حداکثر تعداد تکرار یادآوری برای کار جدید")
        row3.addWidget(self.new_max_reminds_spin)

        lbl_mr = QLabel("حداکثر تکرار:")
        lbl_mr.setFont(QFont("Tahoma", 9))
        row3.addWidget(lbl_mr)

        layout.addLayout(row3)

        lbl_row = QHBoxLayout()
        lbl_row.setContentsMargins(0, 0, 0, 0)
        list_label = QLabel("کارها:")
        list_label.setFont(QFont("Tahoma", 10, QFont.Bold))
        hint = QLabel("(ویرایش: ✏ یا دوبار کلیک)")
        hint.setFont(QFont("Tahoma", 8))
        hint.setStyleSheet("color: #999999;")
        lbl_row.addWidget(hint)
        lbl_row.addStretch()
        lbl_row.addWidget(list_label)
        layout.addLayout(lbl_row)

        self.scroll_area = QScrollArea()
        self.scroll_area.setWidgetResizable(True)
        self.scroll_area.setMinimumHeight(180)
        self.scroll_area.setStyleSheet("""
            QScrollArea {
                border: 1px solid #ccc;
                border-radius: 5px;
                background-color: #ffffff;
            }
        """)

        self.tasks_container = QWidget()
        self.tasks_container.setLayoutDirection(Qt.LeftToRight)
        self.tasks_layout = QVBoxLayout(self.tasks_container)
        self.tasks_layout.setContentsMargins(2, 2, 2, 2)
        self.tasks_layout.setSpacing(2)
        self.tasks_layout.addStretch()
        self.scroll_area.setWidget(self.tasks_container)

        layout.addWidget(self.scroll_area)

        btn_row = QHBoxLayout()
        btn_row.setSpacing(5)

        del_btn = QPushButton("حذف انتخاب‌شده")
        del_btn.setFont(QFont("Tahoma", 9))
        del_btn.setMinimumHeight(26)
        del_btn.setStyleSheet("""
            QPushButton { background-color: #e0e0e0; color: #333; border-radius: 5px; padding: 3px; }
            QPushButton:hover { background-color: #d0d0d0; }
        """)
        del_btn.clicked.connect(self._delete_task)
        btn_row.addWidget(del_btn)

        del_completed_btn = QPushButton("حذف پایان یافته‌ها")
        del_completed_btn.setFont(QFont("Tahoma", 9))
        del_completed_btn.setMinimumHeight(26)
        del_completed_btn.setToolTip("حذف تمام کارهای علامت‌خورده به‌عنوان انجام‌شده")
        del_completed_btn.setStyleSheet("""
            QPushButton { background-color: #f0ad4e; color: white; border-radius: 5px; padding: 3px; }
            QPushButton:hover { background-color: #ec971f; }
        """)
        del_completed_btn.clicked.connect(self._delete_completed_tasks)
        btn_row.addWidget(del_completed_btn)

        del_all_btn = QPushButton("حذف همه")
        del_all_btn.setFont(QFont("Tahoma", 9))
        del_all_btn.setMinimumHeight(26)
        del_all_btn.setStyleSheet("""
            QPushButton { background-color: #d9534f; color: white; border-radius: 5px; padding: 3px; }
            QPushButton:hover { background-color: #c9302c; }
        """)
        del_all_btn.clicked.connect(self._delete_all_tasks)
        btn_row.addWidget(del_all_btn)

        print_btn = QPushButton("🖨 چاپ لیست امروز")
        print_btn.setFont(QFont("Tahoma", 9))
        print_btn.setMinimumHeight(26)
        print_btn.setToolTip("پیش‌نمایش و چاپ لیست کارهای امروز به صورت جدول")
        print_btn.setStyleSheet("""
            QPushButton { background-color: #0078d4; color: white; border-radius: 5px; padding: 3px; }
            QPushButton:hover { background-color: #1a8ae0; }
        """)
        print_btn.clicked.connect(self._print_today_tasks)
        btn_row.addWidget(print_btn)

        layout.addLayout(btn_row)

        self.startup_checkbox = QCheckBox("اجرا هنگام روشن شدن ویندوز")
        self.startup_checkbox.setFont(QFont("Tahoma", 9))
        self.startup_checkbox.setLayoutDirection(Qt.RightToLeft)
        self.startup_checkbox.setChecked(is_startup_enabled())
        self.startup_checkbox.toggled.connect(self._toggle_startup)
        layout.addWidget(self.startup_checkbox)

        dev_label = QLabel("برنامه‌نویس: مهدی زارع  |  MehdiZare@Outlook.com")
        dev_label.setAlignment(Qt.AlignCenter)
        dev_label.setFont(QFont("Tahoma", 8))
        dev_label.setStyleSheet("color: #888888; padding-top: 1px;")
        layout.addWidget(dev_label)

        self.setLayout(layout)

    def _sync_datetime_fields(self):
        try:
            now_j = jdatetime.datetime.now()
            now_g = datetime.now()

            year_str = str(now_j.year)
            idx = self.year_combo.findText(year_str)
            if idx < 0:
                self.year_combo.blockSignals(True)
                self.year_combo.insertItem(0, year_str)
                _style_combo_center(self.year_combo)
                self.year_combo.setCurrentIndex(0)
                self.year_combo.blockSignals(False)
            else:
                self.year_combo.blockSignals(True)
                self.year_combo.setCurrentIndex(idx)
                self.year_combo.blockSignals(False)

            self.month_combo.blockSignals(True)
            self.month_combo.setCurrentIndex(now_j.month - 1)
            self.month_combo.blockSignals(False)

            max_days = days_in_jalali_month(now_j.year, now_j.month)
            current_day = min(now_j.day, max_days)
            self.day_combo.blockSignals(True)
            self.day_combo.clear()
            for d in range(1, max_days + 1):
                self.day_combo.addItem(str(d))
            _style_combo_center(self.day_combo)
            self.day_combo.setCurrentText(str(current_day))
            self.day_combo.blockSignals(False)

            self.hour_spin.blockSignals(True)
            self.hour_spin.setValue(now_g.hour)
            self.hour_spin.blockSignals(False)

            self.minute_spin.blockSignals(True)
            self.minute_spin.setValue(now_g.minute)
            self.minute_spin.blockSignals(False)

            self.date_label.setText(get_jalali_date_string())

            log_info(
                f"فیلدهای تاریخ/ساعت با زمان جاری همگام شد: "
                f"{now_j.year}/{now_j.month:02d}/{now_j.day:02d} "
                f"{now_g.hour:02d}:{now_g.minute:02d}"
            )
        except Exception as e:
            log_error(f"خطا در _sync_datetime_fields: {e}\n{traceback.format_exc()}")

    def showEvent(self, event):
        super().showEvent(event)
        QTimer.singleShot(0, self._sync_datetime_fields)

    def _update_days(self):
        try:
            year = int(self.year_combo.currentText())
            month = self.month_combo.currentIndex() + 1
            max_days = days_in_jalali_month(year, month)
            current = self.day_combo.currentText()
            self.day_combo.blockSignals(True)
            self.day_combo.clear()
            for d in range(1, max_days + 1):
                self.day_combo.addItem(str(d))
            _style_combo_center(self.day_combo)
            if current and current.isdigit() and 1 <= int(current) <= max_days:
                self.day_combo.setCurrentText(current)
            self.day_combo.blockSignals(False)
        except Exception as e:
            log_error(f"خطا در _update_days: {e}")

    def _toggle_startup(self, checked):
        set_startup(checked)

    def _setup_tray(self):
        if not TRAY_AVAILABLE:
            return
        try:
            def on_show(icon, item):
                QTimer.singleShot(0, self._show_main_window)

            def on_exit(icon, item):
                try:
                    icon.stop()
                except Exception:
                    pass
                QTimer.singleShot(0, self._quit_app)

            menu = TrayMenu(
                TrayMenuItem("نمایش برنامه", on_show, default=True),
                TrayMenuItem("خروج", on_exit),
            )

            tray_img = create_default_icon_image(64)
            self.tray_icon = TrayIcon(APP_NAME, tray_img, APP_TITLE, menu)
            threading.Thread(target=self.tray_icon.run, daemon=True).start()
        except Exception as e:
            log_error(f"خطا در راه‌اندازی سینی: {e}")

    def _setup_single_instance_server(self):
        try:
            QLocalServer.removeServer(SINGLE_INSTANCE_KEY)
            self._local_server = QLocalServer(self)
            self._local_server.newConnection.connect(self._handle_new_instance_connection)
            if not self._local_server.listen(SINGLE_INSTANCE_KEY):
                log_error(
                    "خطا در راه‌اندازی سرور single-instance: "
                    f"{self._local_server.errorString()}"
                )
            else:
                log_info("سرور single-instance راه‌اندازی شد")
        except Exception as e:
            log_error(f"خطا در _setup_single_instance_server: {e}")

    def _handle_new_instance_connection(self):
        try:
            conn = self._local_server.nextPendingConnection()
            if conn is None:
                return

            def _read_message():
                try:
                    data = bytes(conn.readAll())
                    if b"SHOW" in data:
                        log_info("نمونه جدید اجرا شد → بالا آوردن پنجره اصلی")
                        self._show_main_window()
                except Exception as e:
                    log_error(f"خطا در خواندن پیام نمونه جدید: {e}")

            conn.readyRead.connect(_read_message)
            conn.disconnected.connect(conn.deleteLater)
        except Exception as e:
            log_error(f"خطا در _handle_new_instance_connection: {e}")

    def _show_main_window(self):
        self._sync_datetime_fields()
        self.showNormal()
        self.activateWindow()
        self.raise_()
        QTimer.singleShot(0, self._sync_datetime_fields)

    def _quit_app(self):
        try:
            if hasattr(self, "_local_server"):
                self._local_server.close()
                QLocalServer.removeServer(SINGLE_INSTANCE_KEY)
        except Exception:
            pass
        try:
            if hasattr(self, "tray_icon"):
                self.tray_icon.stop()
        except Exception:
            pass
        QApplication.quit()

    def closeEvent(self, event):
        event.ignore()
        self.hide()

    def changeEvent(self, event):
        if event.type() == QEvent.WindowStateChange:
            if self.windowState() & Qt.WindowMinimized:
                QTimer.singleShot(0, self.hide)
            else:
                QTimer.singleShot(0, self._sync_datetime_fields)
        super().changeEvent(event)

    def _add_task(self):
        title = self.task_input.text().strip()
        if not title:
            show_message_box(self, QMessageBox.Warning, "خطا", "عنوان کار را وارد کنید.")
            return

        time_str = f"{self.hour_spin.value():02d}:{self.minute_spin.value():02d}"
        year = int(self.year_combo.currentText())
        month = self.month_combo.currentIndex() + 1
        day = int(self.day_combo.currentText())

        try:
            if is_past_jalali_datetime(
                year, month, day,
                self.hour_spin.value(),
                self.minute_spin.value()
            ):
                show_message_box(
                    self, QMessageBox.Warning, "خطا",
                    "گذشته‌ها گذشته به فکر آینده باش"
                )
                return
        except Exception:
            show_message_box(self, QMessageBox.Warning, "خطا", "تاریخ و ساعت معتبر نیست.")
            return

        date_str = f"{year}/{month:02d}/{day:02d}"

        self.tasks.append({
            "id": int(time.time() * 1000),
            "title": title,
            "date": date_str,
            "time": time_str,
            "snooze_until": None,
            "last_triggered": None,
            "done": False,
            "forgotten": False,
            "remind_interval": self.new_remind_interval_spin.value(),
            "max_reminds": self.new_max_reminds_spin.value(),
            "remind_count": 0,
        })
        save_tasks(self.tasks)
        self.task_input.clear()
        self._refresh_task_list()

    def _delete_task(self):
        if self._selected_task_id is None:
            show_message_box(
                self, QMessageBox.Information, "توجه",
                "ابتدا روی یک کار کلیک کنید تا انتخاب شود."
            )
            return
        self.tasks = [t for t in self.tasks if t["id"] != self._selected_task_id]
        self._selected_task_id = None
        save_tasks(self.tasks)
        self._refresh_task_list()

    def _delete_completed_tasks(self):
        completed = [t for t in self.tasks if t.get("done", False)]
        if not completed:
            show_message_box(
                self, QMessageBox.Information, "توجه",
                "هیچ کار تمام شده‌ای برای حذف وجود ندارد."
            )
            return
        reply = show_message_box(
            self, QMessageBox.Question, "تأیید حذف پایان یافته ها",
            f"آیا از حذف {len(completed)} کار مختومه مطمئن هستید؟",
            QMessageBox.Yes | QMessageBox.No,
            QMessageBox.No
        )
        if reply == QMessageBox.Yes:
            self.tasks = [t for t in self.tasks if not t.get("done", False)]
            if self._selected_task_id is not None:
                if not any(t["id"] == self._selected_task_id for t in self.tasks):
                    self._selected_task_id = None
            save_tasks(self.tasks)
            self._refresh_task_list()
            log_info(f"حذف {len(completed)} کار مختومه انجام شد")

    def _delete_all_tasks(self):
        if not self.tasks:
            return
        reply = show_message_box(
            self, QMessageBox.Question, "تأیید حذف",
            "آیا از حذف همه کارها مطمئن هستید؟",
            QMessageBox.Yes | QMessageBox.No,
            QMessageBox.No
        )
        if reply == QMessageBox.Yes:
            self.tasks = []
            self._selected_task_id = None
            save_tasks(self.tasks)
            self._refresh_task_list()

    # ---------- ساخت HTML لیست کارهای امروز (پارامتریک) ----------
    def _build_today_tasks_html(self, font_size_pt=10, padding_px=4,
                                line_height=1.15, title_size_pt=15,
                                date_size_pt=11):
        try:
            print_now_j = jdatetime.datetime.now()
            print_date = (
                f"{print_now_j.year}/{print_now_j.month:02d}/{print_now_j.day:02d}"
            )
            print_time = datetime.now().strftime("%H:%M")
            print_footer_line = (
                f"نرم افزار یادآور آویژه   نسخه {to_persian_digits(APP_VERSION)}    "
                f":زمان چاپ گزارش {to_persian_digits(print_date)} - {to_persian_digits(print_time)}"
            )
        except Exception:
            print_footer_line = (
                f"نرم افزار یادآور آویژه  |  نسخه {to_persian_digits(APP_VERSION)}"
            )

        try:
            today = today_jalali_str()
            today_tasks = [t for t in self.tasks if t.get("date") == today]
            if not today_tasks:
                return None, None

            try:
                today_tasks.sort(key=lambda t: t["time"])
            except Exception:
                pass

            now_j = jdatetime.datetime.now()
            weekday_name = JALALI_WEEKDAYS[now_j.weekday()]
            month_name = JALALI_MONTHS[now_j.month - 1]
            date_line = (
                f"{weekday_name} {to_persian_digits(now_j.day)} "
                f"{month_name} {to_persian_digits(now_j.year)}"
            )

            font_family = VAZIR_FONT_FAMILY or "Tahoma"

            def esc(s):
                return (str(s)
                        .replace("&", "&amp;")
                        .replace("<", "&lt;")
                        .replace(">", "&gt;"))

            v_pad = max(int(padding_px), 1)
            h_pad = max(int(padding_px) - 1, 1)
            cell_style = f"border:1px solid #000; padding:{v_pad}px {h_pad}px;"

            rows_html = ""
            for idx, t in enumerate(today_tasks, start=1):
                if t.get("forgotten", False):
                    status_text = "علی رغم یادآوری مکرر فراموش شد"
                    status_color = "#8a5a00"
                elif t.get("done", False):
                    status_text = "با موفقیت انجام شد"
                    status_color = "#1b7a1b"
                else:
                    status_text = "انجام نشده است"
                    status_color = "#a00000"

                rows_html += (
                    "<tr>"
                    f"<td style='text-align:center; {cell_style} color:{status_color};'>"
                    f"{status_text}</td>"
                    f"<td style='text-align:center; {cell_style}'>"
                    f"{to_persian_digits(t.get('time',''))}</td>"
                    f"<td style='text-align:center; {cell_style}'>"
                    f"{esc(t.get('title',''))}</td>"
                    f"<td style='text-align:center; {cell_style}'>"
                    f"{to_persian_digits(idx)}</td>"
                    "</tr>"
                )

            header_cell_style = (
                f"border:1px solid #000; padding:{v_pad}px {h_pad}px; "
                f"text-align:center; background-color:#d9e7f5; "
                f"font-family:'{font_family}', Tahoma;"
            )

            html = f"""
            <html>
            <head><meta charset="utf-8"></head>
            <body dir="rtl" style="direction:rtl; font-family: '{font_family}', Tahoma; margin:0; padding:0; line-height:{line_height};">
              <h1 style="text-align:center; font-size:{title_size_pt}pt; margin:0 0 2px 0;
                         font-family:'{font_family}', Tahoma;">لیست کارهای امروز</h1>
              <h2 style="text-align:center; font-size:{date_size_pt}pt; margin:0 0 10px 0;
                         font-weight:normal; font-family:'{font_family}', Tahoma;">
                {date_line}
              </h2>
              <table width="100%" cellspacing="0" cellpadding="3" border="1"
                     style="border-collapse:collapse; border:1px solid #000;
                            font-size:{font_size_pt}pt; font-family:'{font_family}', Tahoma;
                            line-height:{line_height}; table-layout:fixed;">
                <thead>
                  <tr>
                    <th width="30%" style="{header_cell_style}">وضعیت</th>
                    <th width="15%" style="{header_cell_style}">ساعت مقرر</th>
                    <th width="47%" style="{header_cell_style}">عنوان کار</th>
                    <th width="8%"  style="{header_cell_style}">ردیف</th>
                  </tr>
                </thead>
                <tbody>
                  {rows_html}
                </tbody>
              </table>

              <div dir="rtl"
                   style="direction:rtl; width:86%; max-width:680px;
                          margin:34px auto 0 auto;
                          text-align:center; font-family:'{font_family}', Tahoma;
                          font-size:12pt; font-weight:bold; overflow:hidden;">
                <table dir="rtl" align="center" cellspacing="0" cellpadding="0"
                       style="direction:rtl; width:100%; max-width:680px;
                              border-collapse:collapse; margin:0 auto;
                              text-align:center; table-layout:fixed;">
                  <tr>
                    <td dir="rtl"
                        style="direction:rtl; unicode-bidi:plaintext;
                               text-align:center; white-space:nowrap;
                               overflow:hidden; padding:0;">
                      &#x200F;مشخصات برنامه ریز
                    </td>
                  </tr>
                  <tr>
                    <td dir="rtl"
                        style="direction:rtl; unicode-bidi:plaintext;
                               text-align:center; white-space:nowrap;
                               overflow:hidden; padding:20px 0 0 0;">
                      &#x200F;امضاء
                    </td>
                  </tr>
                </table>
              </div>
            </body>
            </html>
            """
            return html, print_footer_line
        except Exception as e:
            log_error(f"خطا در _build_today_tasks_html: {e}\n{traceback.format_exc()}")
            return None, None

    # ---------- اعمال اجباری RTL روی متن ----------
    def _force_rtl_document(self, doc, date_line=""):
        try:
            try:
                option = QTextOption()
                option.setTextDirection(Qt.RightToLeft)
                doc.setDefaultTextOption(option)
            except Exception as e:
                log_error(f"خطا در setDefaultTextOption: {e}")

            try:
                block = doc.begin()
                while block.isValid():
                    text = block.text().strip()
                    clean_text = (text
                                  .replace("\u202b", "")
                                  .replace("\u202c", "")
                                  .replace("\u200f", "")
                                  .replace("\u200e", "")
                                  .strip())

                    bf = block.blockFormat()
                    bf.setLayoutDirection(Qt.RightToLeft)

                    is_title = (clean_text == "لیست کارهای امروز")
                    is_date = bool(date_line) and (clean_text == date_line)
                    is_footer_1 = clean_text.startswith("مشخصات برنامه ریز")
                    is_footer_2 = clean_text.startswith("امضاء")

                    if is_title or is_date or is_footer_1 or is_footer_2:
                        bf.setAlignment(Qt.AlignCenter)
                        bf.setLayoutDirection(Qt.RightToLeft)

                    cur = QTextCursor(block)
                    cur.setBlockFormat(bf)
                    block = block.next()
            except Exception as e:
                log_error(f"خطا در پیمایش بلاک‌ها: {e}")

            try:
                root = doc.rootFrame()
                ff = root.frameFormat()
                ff.setLayoutDirection(Qt.RightToLeft)
                root.setFrameFormat(ff)
            except Exception as e:
                log_error(f"خطا در تنظیم فریم ریشه: {e}")

        except Exception as e:
            log_error(f"خطا در _force_rtl_document: {e}\n{traceback.format_exc()}")

    # ---------- اعمال فونت وزیر روی همه کاراکترهای سند ----------
    def _apply_font_to_document(self, doc):
        family = VAZIR_FONT_FAMILY
        if not family:
            return
        try:
            cursor = QTextCursor(doc)
            cursor.beginEditBlock()
            cursor.select(QTextCursor.Document)
            char_fmt = QTextCharFormat()
            char_fmt.setFontFamily(family)
            cursor.mergeCharFormat(char_fmt)
            cursor.endEditBlock()
            log_info(f"فونت '{family}' روی سند چاپ اعمال شد")
        except Exception as e:
            log_error(f"خطا در _apply_font_to_document: {e}")

    # ---------- رندر سند روی چاپگر (با فوتر پایین صفحه و تطبیق خودکار) ----------
    def _render_print_document(self, printer, html="", date_line="", footer_text="",
                               rebuild_html=False):
        try:
            font_family = VAZIR_FONT_FAMILY or "Tahoma"

            # ----- ابعاد ناحیه قابل چاپ (به پوینت) -----
            try:
                page_layout = printer.pageLayout()
                page_size_pt = page_layout.pageSize().size(QPageSize.Point)
                margins_pt = page_layout.margins(QPageLayout.Point)
                printable_w_pt = (page_size_pt.width()
                                  - margins_pt.left() - margins_pt.right())
                printable_h_pt = (page_size_pt.height()
                                  - margins_pt.top() - margins_pt.bottom())
                log_info(
                    f"صفحه: {page_size_pt.width():.1f}x{page_size_pt.height():.1f}pt | "
                    f"حاشیه: L={margins_pt.left():.1f} R={margins_pt.right():.1f} "
                    f"T={margins_pt.top():.1f} B={margins_pt.bottom():.1f} | "
                    f"قابل چاپ: {printable_w_pt:.1f}x{printable_h_pt:.1f}pt"
                )
            except Exception as e:
                log_error(f"خطا در محاسبه ابعاد ناحیه چاپ: {e}")
                printable_w_pt = 523.0
                printable_h_pt = 770.0

            # ----- حاشیه داخلی -----
            inner_margin_pt = 40.0        # ~ 14mm از هر طرف چپ و راست
            top_inner_margin_pt = 6.0
            footer_h_pt = 32.0            # ~ 11mm ارتفاع ناحیه فوتر

            content_w_pt = max(printable_w_pt - 2 * inner_margin_pt, 100.0)
            content_h_pt = max(printable_h_pt - top_inner_margin_pt - footer_h_pt, 100.0)

            # ----- اگر rebuild_html: بهترین اندازه فونت را پیدا کن -----
            if rebuild_html:
                candidates = [
                    (10, 4, 1.15, 15, 11),   # اصلی
                    (9,  3, 1.10, 14, 10),
                    (8,  3, 1.05, 13, 9),
                    (7,  2, 1.00, 12, 9),
                    (6,  2, 0.95, 11, 8),
                    (5,  1, 0.90, 10, 8),
                    (4,  1, 0.85, 9,  7),
                ]
                chosen_html = None
                last_html = None
                last_footer = footer_text

                for font_size, padding, line_h, title_sz, date_sz in candidates:
                    res = self._build_today_tasks_html(
                        font_size_pt=font_size, padding_px=padding,
                        line_height=line_h, title_size_pt=title_sz,
                        date_size_pt=date_sz,
                    )
                    if res is None or res[0] is None:
                        return
                    cand_html, cand_footer = res
                    last_html = cand_html
                    last_footer = cand_footer

                    test_doc = QTextDocument()
                    test_doc.setHtml(cand_html)
                    try:
                        test_doc.setDefaultFont(QFont(font_family, 10))
                    except Exception:
                        pass
                    self._force_rtl_document(test_doc, date_line=date_line)
                    self._apply_font_to_document(test_doc)
                    test_doc.setDocumentMargin(0)
                    test_doc.setPageSize(QSizeF(content_w_pt, content_h_pt))

                    if test_doc.pageCount() <= 1:
                        chosen_html = cand_html
                        footer_text = cand_footer
                        log_info(
                            f"چاپ: فونت {font_size}pt / پدینگ {padding}px "
                            f"برای جاگیری در یک صفحه انتخاب شد"
                        )
                        break

                if chosen_html is None:
                    chosen_html = last_html
                    footer_text = last_footer
                    log_info("چاپ: حتی کوچک‌ترین فونت هم در یک صفحه جا نشد")

                html = chosen_html

            # ----- رندر سند نهایی -----
            doc = QTextDocument()
            doc.setHtml(html)
            try:
                doc.setDefaultFont(QFont(font_family, 10))
            except Exception:
                pass

            self._force_rtl_document(doc, date_line=date_line)
            self._apply_font_to_document(doc)

            doc.setDocumentMargin(0)
            doc.setPageSize(QSizeF(content_w_pt, content_h_pt))

            resolution = printer.resolution()
            if resolution <= 0:
                resolution = 300
            scale = resolution / 72.0

            try:
                page_rect_px = printer.pageRect()
                offset_x_px = page_rect_px.left()
                offset_y_px = page_rect_px.top()
            except Exception:
                offset_x_px = 0
                offset_y_px = 0

            painter = QPainter()
            if not painter.begin(printer):
                log_error("خطا در باز کردن QPainter روی چاپگر")
                return

            try:
                page_count = doc.pageCount()
                if page_count < 1:
                    page_count = 1

                for page_num in range(page_count):
                    if page_num > 0:
                        printer.newPage()

                    # ------- محتوا -------
                    painter.save()
                    painter.resetTransform()
                    painter.translate(offset_x_px, offset_y_px)
                    painter.scale(scale, scale)
                    painter.translate(inner_margin_pt, top_inner_margin_pt)
                    clip = QRectF(0.0, 0.0, content_w_pt, content_h_pt)
                    painter.setClipRect(clip)
                    doc.drawContents(painter, clip)
                    painter.restore()

                    # ------- فوتر پایین صفحه -------
                    if footer_text:
                        painter.save()
                        painter.resetTransform()
                        painter.translate(offset_x_px, offset_y_px)
                        painter.scale(scale, scale)
                        painter.setClipping(False)

                        footer_rect = QRectF(
                            inner_margin_pt,
                            top_inner_margin_pt + content_h_pt + 3.0,
                            content_w_pt,
                            footer_h_pt - 5.0
                        )

                        try:
                            # *** نکته کلیدی: pixelSize به جای pointSize ***
                            base_px = 6

                            footer_font = QFont(font_family)
                            footer_font.setPixelSize(base_px)
                            footer_font.setWeight(QFont.Normal)

                            footer_doc = QTextDocument()
                            footer_doc.setDocumentMargin(0)
                            footer_doc.setDefaultFont(footer_font)
                            footer_doc.setPageSize(
                                QSizeF(footer_rect.width(), footer_rect.height())
                            )
                            footer_doc.setTextWidth(footer_rect.width())

                            footer_html = (
                                f"<div dir='rtl' "
                                f"style='direction:rtl; text-align:center; "
                                f"font-family:\"{font_family}\", Tahoma; "
                                f"font-size:{base_px}px; font-weight:normal; "
                                f"color:#333333; line-height:1.35; "
                                f"white-space:normal; word-wrap:break-word;'>"
                                f"{footer_text}"
                                f"</div>"
                            )
                            footer_doc.setHtml(footer_html)
                            footer_doc.setTextWidth(footer_rect.width())

                            doc_h = footer_doc.size().height()

                            if doc_h > footer_rect.height():
                                for fs in (5, 4, 3):
                                    footer_font.setPixelSize(fs)
                                    footer_doc.setDefaultFont(footer_font)
                                    footer_doc.setHtml(
                                        footer_html.replace(
                                            f"font-size:{base_px}px",
                                            f"font-size:{fs}px"
                                        )
                                    )
                                    footer_doc.setTextWidth(footer_rect.width())
                                    doc_h = footer_doc.size().height()
                                    if doc_h <= footer_rect.height():
                                        break

                            y_offset = footer_rect.top() + max(
                                0.0, (footer_rect.height() - doc_h) / 2.0
                            )

                            painter.translate(footer_rect.left(), y_offset)
                            painter.setClipRect(
                                QRectF(0, 0, footer_rect.width(), footer_rect.height())
                            )
                            footer_doc.drawContents(
                                painter,
                                QRectF(0, 0, footer_rect.width(), footer_rect.height())
                            )
                        except Exception as e:
                            log_error(f"خطا در رسم فوتر: {e}")
                        finally:
                            painter.restore()
            finally:
                painter.end()
        except Exception as e:
            log_error(
                f"خطا در _render_print_document: {e}\n{traceback.format_exc()}"
            )

    # ---------- پیش‌نمایش و چاپ لیست کارهای امروز ----------
    def _print_today_tasks(self):
        try:
            today = today_jalali_str()
            today_tasks = [t for t in self.tasks if t.get("date") == today]
            if not today_tasks:
                show_message_box(
                    self, QMessageBox.Information, "توجه",
                    "هیچ کاری برای امروز ثبت نشده است."
                )
                return

            now_j = jdatetime.datetime.now()
            weekday_name = JALALI_WEEKDAYS[now_j.weekday()]
            month_name = JALALI_MONTHS[now_j.month - 1]
            date_line = (
                f"{weekday_name} {to_persian_digits(now_j.day)} "
                f"{month_name} {to_persian_digits(now_j.year)}"
            )

            printer = QPrinter(QPrinter.HighResolution)
            try:
                printer.setPageSize(QPageSize(QPageSize.A4))
            except Exception:
                try:
                    printer.setPageSize(QPrinter.A4)
                except Exception:
                    pass
            printer.setOrientation(QPrinter.Portrait)
            printer.setDocName("لیست کارهای امروز")

            preview = QPrintPreviewDialog(printer, self)
            preview.setWindowTitle("پیش‌نمایش چاپ لیست کارهای امروز")
            try:
                preview.setLayoutDirection(Qt.RightToLeft)
            except Exception:
                pass
            try:
                preview.resize(900, 720)
            except Exception:
                pass

            # HTML در خود render به‌صورت تطبیقی ساخته می‌شود
            preview.paintRequested.connect(
                lambda p: self._render_print_document(
                    p, date_line=date_line, rebuild_html=True
                )
            )

            log_info("نمایش دیالوگ پیش‌نمایش چاپ لیست کارهای امروز")
            preview.exec_()
            log_info("دیالوگ پیش‌نمایش چاپ بسته شد")
        except Exception as e:
            log_error(f"خطا در _print_today_tasks: {e}\n{traceback.format_exc()}")
            show_message_box(
                self, QMessageBox.Critical, "خطا",
                f"خطا در پیش‌نمایش چاپ لیست کارها:\n{e}"
            )

    def _clear_rows(self):
        while self.tasks_layout.count() > 1:
            item = self.tasks_layout.takeAt(0)
            w = item.widget()
            if w is not None:
                w.setParent(None)
                w.deleteLater()

    def _update_task_setting(self, task_id, key, value):
        try:
            for t in self.tasks:
                if t["id"] == task_id:
                    t[key] = value
                    break
            save_tasks(self.tasks)
        except Exception as e:
            log_error(f"خطا در _update_task_setting: {e}")

    def _refresh_task_list(self):
        try:
            self._clear_rows()

            today = today_jalali_str()
            try:
                self.tasks.sort(key=lambda t: (t.get("date", today), t["time"]))
            except Exception:
                pass

            for task in self.tasks:
                is_done = bool(task.get("done", False))
                is_forgotten = bool(task.get("forgotten", False))
                task_date = task.get("date", today)
                is_today = (task_date == today)
                is_selected = (task["id"] == self._selected_task_id)

                row = QFrame()
                row.setFixedHeight(46)
                row.setLayoutDirection(Qt.LeftToRight)

                if is_selected and is_forgotten:
                    row.setStyleSheet(
                        "QFrame { background-color: #ffcc80; "
                        "border-bottom: 1px solid #e5e5e5; }"
                    )
                elif is_selected:
                    row.setStyleSheet(
                        "QFrame { background-color: #cfe6ff; "
                        "border-bottom: 1px solid #e5e5e5; }"
                    )
                elif is_forgotten:
                    row.setStyleSheet(
                        "QFrame { background-color: #ffe0b2; "
                        "border-bottom: 1px solid #e5e5e5; }"
                    )
                else:
                    row.setStyleSheet(
                        "QFrame { background-color: transparent; "
                        "border-bottom: 1px solid #e5e5e5; }"
                    )

                rl = QHBoxLayout(row)
                rl.setContentsMargins(4, 2, 4, 2)
                rl.setSpacing(4)

                btn = QPushButton()
                btn.setFixedSize(72, 28)
                btn.setFont(QFont("Tahoma", 9, QFont.Bold))
                btn.setCursor(Qt.PointingHandCursor)

                if is_forgotten:
                    btn.setText("فراموش")
                    btn.setToolTip("برای بازگردانی این کار کلیک کنید")
                    btn.setStyleSheet("""
                        QPushButton {
                            background-color: #f0ad4e;
                            color: #ffffff;
                            border: none;
                            border-radius: 5px;
                        }
                        QPushButton:hover { background-color: #ec971f; }
                    """)
                elif is_done:
                    btn.setText(" پایان یافته")
                    btn.setStyleSheet("""
                        QPushButton {
                            background-color: #d9534f;
                            color: #ffffff;
                            border: none;
                            border-radius: 5px;
                        }
                        QPushButton:hover { background-color: #c9302c; }
                    """)
                else:
                    btn.setText("انجام شد")
                    btn.setStyleSheet("""
                        QPushButton {
                            background-color: #28a745;
                            color: #ffffff;
                            border: none;
                            border-radius: 5px;
                        }
                        QPushButton:hover { background-color: #218838; }
                    """)

                tid = task["id"]
                btn.clicked.connect(lambda checked=False, t=tid: self._toggle_done(t))
                rl.addWidget(btn, 0)

                edit_btn = QPushButton("✏")
                edit_btn.setFixedSize(28, 28)
                edit_btn.setFont(QFont("Tahoma", 10))
                edit_btn.setCursor(Qt.PointingHandCursor)
                edit_btn.setToolTip("ویرایش کار")
                edit_btn.setStyleSheet("""
                    QPushButton {
                        background-color: transparent;
                        color: #0078d4;
                        border: 1px solid #cccccc;
                        border-radius: 5px;
                    }
                    QPushButton:hover { background-color: #e8f2fb; border: 1px solid #0078d4; }
                """)
                edit_btn.clicked.connect(lambda checked=False, t=tid: self._edit_task(t))
                rl.addWidget(edit_btn, 0)

                interval_spin = QSpinBox()
                interval_spin.setRange(1, 120)
                interval_spin.setValue(int(task.get("remind_interval", DEFAULT_REMIND_INTERVAL)))
                interval_spin.setSuffix(" د")
                interval_spin.setFixedWidth(52)
                interval_spin.setFixedHeight(24)
                interval_spin.setFont(QFont("Tahoma", 8))
                interval_spin.setLayoutDirection(Qt.LeftToRight)
                interval_spin.setAlignment(Qt.AlignCenter)
                interval_spin.setToolTip("فاصله یادآوری (دقیقه)")
                interval_spin.valueChanged.connect(
                    lambda v, t=tid: self._update_task_setting(t, "remind_interval", int(v))
                )
                rl.addWidget(interval_spin, 0)

                max_spin = QSpinBox()
                max_spin.setRange(1, 20)
                max_spin.setValue(int(task.get("max_reminds", DEFAULT_MAX_REMINDS)))
                max_spin.setSuffix(" ب")
                max_spin.setFixedWidth(46)
                max_spin.setFixedHeight(24)
                max_spin.setFont(QFont("Tahoma", 8))
                max_spin.setLayoutDirection(Qt.LeftToRight)
                max_spin.setAlignment(Qt.AlignCenter)
                max_spin.setToolTip("حداکثر تعداد تکرار یادآوری (بار)")
                max_spin.valueChanged.connect(
                    lambda v, t=tid: self._update_task_setting(t, "max_reminds", int(v))
                )
                rl.addWidget(max_spin, 0)

                rl.addStretch(1)

                time_lbl = QLabel(f"ساعت {task['time']}")
                time_lbl.setFont(QFont("Tahoma", 9))
                time_lbl.setLayoutDirection(Qt.RightToLeft)
                time_lbl.setAlignment(Qt.AlignCenter)
                time_lbl.setAttribute(Qt.WA_TransparentForMouseEvents, True)
                if is_forgotten:
                    time_lbl.setStyleSheet("color: #8a5a00; background: transparent;")
                elif is_done:
                    time_lbl.setStyleSheet("color: #b0b0b0; background: transparent;")
                elif not is_today:
                    time_lbl.setStyleSheet("color: #00008b; background: transparent;")
                else:
                    time_lbl.setStyleSheet("color: #666666; background: transparent;")
                rl.addWidget(time_lbl, 0)

                sep1 = QLabel("|")
                sep1.setFont(QFont("Tahoma", 9))
                sep1.setStyleSheet("color: #cccccc; background: transparent;")
                sep1.setAttribute(Qt.WA_TransparentForMouseEvents, True)
                rl.addWidget(sep1, 0)

                date_display = format_jalali_date_short(task_date)
                date_lbl = QLabel(date_display)
                date_lbl.setFont(QFont("Tahoma", 9))
                date_lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
                date_lbl.setLayoutDirection(Qt.RightToLeft)
                date_lbl.setAttribute(Qt.WA_TransparentForMouseEvents, True)
                if is_forgotten:
                    date_lbl.setStyleSheet("color: #8a5a00; background: transparent;")
                elif is_done:
                    date_lbl.setStyleSheet("color: #b0b0b0; background: transparent;")
                elif not is_today:
                    date_lbl.setStyleSheet("color: #00008b; background: transparent;")
                else:
                    date_lbl.setStyleSheet("color: #666666; background: transparent;")
                rl.addWidget(date_lbl, 0)

                sep2 = QLabel("|")
                sep2.setFont(QFont("Tahoma", 9))
                sep2.setStyleSheet("color: #cccccc; background: transparent;")
                sep2.setAttribute(Qt.WA_TransparentForMouseEvents, True)
                rl.addWidget(sep2, 0)

                if is_forgotten:
                    color = "#8a5a00"
                elif is_done:
                    color = "#b0b0b0"
                elif not is_today:
                    color = "#00008b"
                else:
                    color = "#000000"

                title_text = task["title"]
                if is_forgotten:
                    title_text = f"{title_text} — فراموش شد"

                title_lbl = QLabel(title_text)
                title_lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
                title_lbl.setLayoutDirection(Qt.RightToLeft)
                title_lbl.setWordWrap(False)
                title_lbl.setSizePolicy(QSizePolicy.Maximum, QSizePolicy.Preferred)
                title_lbl.setAttribute(Qt.WA_TransparentForMouseEvents, True)

                try:
                    scroll_w = self.scroll_area.viewport().width()
                    if scroll_w <= 50:
                        scroll_w = self.width() - 30
                except Exception:
                    scroll_w = 560
                available_w = max(scroll_w - 380 - 28, 60)

                base_font = QFont("Tahoma", 10, QFont.Bold)
                fitted_font = QFont(base_font)
                fm = QFontMetrics(fitted_font)
                while (fitted_font.pointSize() > 7
                       and fm.horizontalAdvance(title_text) > available_w):
                    fitted_font.setPointSize(fitted_font.pointSize() - 1)
                    fm = QFontMetrics(fitted_font)

                fitted_font.setStrikeOut(is_done)
                title_lbl.setFont(fitted_font)

                if fm.horizontalAdvance(title_text) > available_w:
                    title_lbl.setText(
                        fm.elidedText(title_text, Qt.ElideRight, available_w)
                    )

                title_lbl.setStyleSheet(
                    f"color: {color}; background: transparent;"
                )
                rl.addWidget(title_lbl, 0)

                if is_forgotten:
                    icon_char = "❌"
                elif is_today:
                    icon_char = "⏰"
                else:
                    icon_char = "📅"

                icon_lbl = QLabel(icon_char)
                icon_lbl.setFont(QFont("Tahoma", 11))
                icon_lbl.setAlignment(Qt.AlignRight | Qt.AlignVCenter)
                icon_lbl.setLayoutDirection(Qt.RightToLeft)
                icon_lbl.setAttribute(Qt.WA_TransparentForMouseEvents, True)
                icon_lbl.setFixedWidth(22)
                icon_lbl.setStyleSheet(
                    f"color: {color}; background: transparent;"
                )
                rl.addWidget(icon_lbl, 0)

                row.mousePressEvent = lambda ev, t=tid: self._on_row_clicked(t)
                row.mouseDoubleClickEvent = lambda ev, t=tid: self._edit_task(t)

                self.tasks_layout.insertWidget(self.tasks_layout.count() - 1, row)

            self.date_label.setText(get_jalali_date_string())
        except Exception as e:
            log_error(f"خطا در _refresh_task_list: {e}\n{traceback.format_exc()}")

    def _on_row_clicked(self, task_id):
        self._selected_task_id = task_id
        self._refresh_task_list()

    def _edit_task(self, task_id):
        log_info(f"_edit_task called: task_id={task_id}")
        task = None
        for t in self.tasks:
            if t["id"] == task_id:
                task = t
                break
        if task is None:
            log_error(f"_edit_task: task با id={task_id} پیدا نشد")
            return

        dlg = EditTaskDialog(task, self)
        if dlg.exec_() == QDialog.Accepted:
            task["title"] = dlg.new_title
            task["date"] = dlg.new_date
            task["time"] = dlg.new_time
            task["remind_interval"] = dlg.new_remind_interval
            task["max_reminds"] = dlg.new_max_reminds
            task["remind_count"] = 0
            task["last_triggered"] = None
            task["snooze_until"] = None
            task["forgotten"] = False
            save_tasks(self.tasks)
            self._refresh_task_list()
            log_info(
                f"_edit_task: ذخیره شد → {task['title']} @ {task['date']} {task['time']}"
            )

    def _toggle_done(self, task_id):
        log_info(f"_toggle_done called: task_id={task_id}")
        try:
            found = False
            for t in self.tasks:
                if t["id"] == task_id:
                    if t.get("forgotten", False):
                        t["forgotten"] = False
                        t["remind_count"] = 0
                        t["snooze_until"] = None
                    else:
                        t["done"] = not t.get("done", False)
                        if t["done"]:
                            t["forgotten"] = False
                            t["snooze_until"] = None
                    found = True
                    break
            if not found:
                log_info("   task_id not found in tasks")
                return
            save_tasks(self.tasks)
            QTimer.singleShot(0, self._refresh_task_list)
        except Exception as e:
            log_error(f"خطا در _toggle_done: {e}\n{traceback.format_exc()}")

    def _show_today_tasks_notification(self):
        try:
            today = today_jalali_str()
            today_tasks = [
                t for t in self.tasks
                if t.get("date") == today
                and not t.get("done", False)
                and not t.get("forgotten", False)
            ]
            if not today_tasks:
                log_info("اعلان امروز: هیچ کار انجام‌نشده‌ای برای امروز وجود ندارد")
                return

            today_tasks.sort(key=lambda t: t["time"])
            log_info(f"نمایش اعلان امروز با {len(today_tasks)} کار")

            notif = TodayTasksNotification(today_tasks)
            self._active_notifications.append(notif)
            notif.closed.connect(lambda n=notif: self._cleanup(n))
            notif.show_notification()
        except Exception as e:
            log_error(f"خطا در اعلان کارهای امروز: {e}\n{traceback.format_exc()}")

    def _start_scheduler(self):
        self.scheduler_timer = QTimer()
        self.scheduler_timer.timeout.connect(self._check_tasks)
        self.scheduler_timer.start(10000)
        QTimer.singleShot(1500, self._check_tasks)

    def _check_tasks(self):
        try:
            now = datetime.now()
            today = today_jalali_str()
            tasks = load_tasks()
            changed = False

            for task in tasks:
                if task.get("done") or task.get("forgotten"):
                    continue

                trigger = False

                if task.get("snooze_until"):
                    try:
                        snooze_dt = datetime.fromisoformat(task["snooze_until"])
                        if now >= snooze_dt:
                            trigger = True
                            task["snooze_until"] = None
                            changed = True
                            log_info(f"Snooze trigger: {task['title']}")
                    except Exception:
                        task["snooze_until"] = None
                        changed = True

                elif task.get("date") == today:
                    try:
                        hh, mm = map(int, task["time"].split(":"))
                        task_dt = now.replace(hour=hh, minute=mm, second=0, microsecond=0)
                        if task_dt <= now < task_dt + timedelta(minutes=5):
                            trigger_key = f"{today}|{task['time']}"
                            if task.get("last_triggered") != trigger_key:
                                trigger = True
                                task["last_triggered"] = trigger_key
                                changed = True
                                log_info(f"Time trigger: {task['title']} @ {task['time']}")
                    except Exception as e:
                        log_error(f"خطا در بررسی کار: {e}")

                if trigger:
                    self._show_notification(
                        task["title"], task["time"], task["id"],
                        remind_count=int(task.get("remind_count", 0)),
                        max_reminds=int(task.get("max_reminds", DEFAULT_MAX_REMINDS)),
                        remind_interval=int(task.get("remind_interval", DEFAULT_REMIND_INTERVAL)),
                    )

            if changed:
                save_tasks(tasks)
        except Exception as e:
            log_error(f"خطا کلی در _check_tasks: {e}\n{traceback.format_exc()}")

    def _show_notification(self, title, time_str, task_id,
                           remind_count=0,
                           max_reminds=DEFAULT_MAX_REMINDS,
                           remind_interval=DEFAULT_REMIND_INTERVAL):
        log_info(
            f"_show_notification called for: {title} @ {time_str} "
            f"(count={remind_count}/{max_reminds}, interval={remind_interval})"
        )
        play_notification_sound()
        try:
            notif = NotificationWindow(
                title, time_str, task_id,
                remind_count=remind_count,
                max_reminds=max_reminds,
                remind_interval=remind_interval,
            )
            self._active_notifications.append(notif)
            notif.closed.connect(lambda n=notif: self._cleanup(n))
            notif.show_notification()
            log_info(f"Notification shown, list size = {len(self._active_notifications)}")
        except Exception as e:
            log_error(f"خطا در نمایش اعلان: {e}\n{traceback.format_exc()}")

    def _cleanup(self, notif):
        if notif in self._active_notifications:
            self._active_notifications.remove(notif)


def main():
    try:
        app = QApplication(sys.argv)
        app.setLayoutDirection(Qt.LeftToRight)
        app.setQuitOnLastWindowClosed(False)
        app.setApplicationName(APP_NAME)
        app.setApplicationDisplayName(APP_TITLE)

        load_vazir_font()

        if activate_existing_instance():
            log_info("نمونه دیگری در حال اجراست؛ پیام SHOW ارسال شد و خارج می‌شویم.")
            sys.exit(0)

        window = ReminderApp()
        app._reminder_window = window

        if "--hidden" not in sys.argv:
            window.show()

        sys.exit(app.exec_())
    except Exception as e:
        log_error("خطای کلی در main:\n" + traceback.format_exc())
        try:
            show_message_box(None, QMessageBox.Critical, "خطا", f"خطا در اجرای برنامه:\n{e}")
        except Exception:
            pass
        sys.exit(1)


if __name__ == "__main__":
    main()نرم افزار یادآور ویندوزی رایگان به زبان فارسی
