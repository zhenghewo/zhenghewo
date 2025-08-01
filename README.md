```python
import csv
import os
from reportlab.lib.pagesizes import letter, A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image, Table, TableStyle, PageBreak
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import inch, mm, cm
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from datetime import datetime

# 自定义页面尺寸：宽度1900点，高度按比例计算（基于A4比例）
CUSTOM_PAGE_SIZE = (1900, 1900 * (842/595))  # 宽度1900点，高度约2685点

def csv_to_pdf(input_csv, output_pdf, title, intro_text, image_path):
    """
    将包含标题、查询条件和表格数据的CSV文件转换为精美的PDF报告
    
    参数:
    input_csv (str): 输入CSV文件路径
    output_pdf (str): 输出PDF文件路径
    title (str): PDF主标题
    intro_text (str): 介绍性文本
    image_path (str): 固定图片路径
    """
    
    # 创建PDF文档 - 使用自定义页面尺寸
    doc = SimpleDocTemplate(
        output_pdf,
        pagesize=CUSTOM_PAGE_SIZE,  # 使用自定义宽度
        rightMargin=72,
        leftMargin=72,
        topMargin=72,
        bottomMargin=36
    )
    
    # 注册中文字体
    try:
        # 尝试注册常见中文字体
        font_paths = [
            'simsun.ttc', 'STSONG.TTF', 'msyh.ttc', 'simhei.ttf',
            '/System/Library/Fonts/PingFang.ttc',  # macOS
            '/usr/share/fonts/truetype/droid/DroidSansFallbackFull.ttf'  # Linux
        ]
        
        registered = False
        for font_path in font_paths:
            try:
                font_name = os.path.splitext(os.path.basename(font_path))[0]
                pdfmetrics.registerFont(TTFont(font_name, font_path))
                print(f"使用字体: {font_name}")
                registered = True
                break
            except:
                continue
        
        if not registered:
            # 回退到英文字体
            font_name = 'Helvetica'
            print("使用默认英文字体")
    except:
        font_name = 'Helvetica'
    
    # 获取样式表
    styles = getSampleStyleSheet()
    
    # 自定义样式 - 针对宽页面优化
    title_style = ParagraphStyle(
        'TitleStyle',
        parent=styles['Heading1'],
        fontName=font_name,
        fontSize=32,  # 增大字号
        alignment=1,  # 居中
        spaceAfter=24,  # 增加间距
        textColor=colors.HexColor('#2C3E50'),
        leading=36
    )
    
    subtitle_style = ParagraphStyle(
        'SubtitleStyle',
        parent=styles['Heading2'],
        fontName=font_name,
        fontSize=24,  # 增大字号
        alignment=0,  # 左对齐
        spaceAfter=12,
        textColor=colors.HexColor('#2980B9'),
        leading=28
    )
    
    intro_style = ParagraphStyle(
        'IntroStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=16,  # 增大字号
        spaceAfter=24,
        textColor=colors.HexColor('#34495E'),
        leading=22
    )
    
    condition_style = ParagraphStyle(
        'ConditionStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=14,  # 增大字号
        spaceAfter=12,
        textColor=colors.HexColor('#7F8C8D'),
        leading=20
    )
    
    footer_style = ParagraphStyle(
        'FooterStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=12,
        textColor=colors.HexColor('#95A5A6'),
        alignment=2  # 右对齐
    )
    
    # 创建内容列表
    story = []
    
    # ==================== 封面页 ====================
    # 添加封面图片（使用全宽度）
    if os.path.exists(image_path):
        # 计算图片高度以保持比例
        img_width = CUSTOM_PAGE_SIZE[0] - 144  # 减去左右边距
        img_height = img_width * 0.4  # 高度为宽度的40%
        
        img = Image(image_path, width=img_width, height=img_height)
        img.hAlign = 'CENTER'
        story.append(img)
        story.append(Spacer(1, 0.5*inch))
    
    # 添加主标题
    story.append(Paragraph(title, title_style))
    story.append(Spacer(1, 0.5*inch))
    
    # 添加介绍文本
    intro_paragraphs = intro_text.split('\n')
    for para in intro_paragraphs:
        if para.strip():
            story.append(Paragraph(para.strip(), intro_style))
    
    # 添加报告信息
    story.append(Spacer(1, 1*inch))
    report_date = datetime.now().strftime("%Y年%m月%d日 %H:%M")
    story.append(Paragraph(f"报告生成时间: {report_date}", condition_style))
    story.append(Paragraph(f"数据来源: {os.path.basename(input_csv)}", condition_style))
    
    story.append(PageBreak())  # 封面页结束
    
    # ==================== 内容页 ====================
    # 读取CSV数据
    report_title = ""
    conditions = []
    header = []
    data = []
    
    with open(input_csv, 'r', encoding='utf-8') as csvfile:
        csv_reader = csv.reader(csvfile)
        
        # 读取报告标题（第一行）
        try:
            first_row = next(csv_reader)
            if first_row:
                report_title = first_row[0].strip('"')
        except StopIteration:
            pass
        
        # 读取查询条件（直到遇到空行或表头）
        for row in csv_reader:
            if not row:  # 跳过空行
                continue
            # 检查是否是表头（包含多个列）
            if len(row) > 1 and any(field.strip() for field in row):
                header = [field.strip('"') for field in row]
                break
            if row[0].strip():
                conditions.append(row[0].strip('"'))
        
        # 读取数据行
        for row in csv_reader:
            if any(field.strip() for field in row):  # 跳过空行
                cleaned_row = [field.strip('"') for field in row]
                data.append(cleaned_row)
    
    # 添加报告标题
    if report_title:
        story.append(Paragraph(report_title, subtitle_style))
        story.append(Spacer(1, 0.3*inch))
    
    # 添加查询条件
    if conditions:
        conditions_html = "<br/>".join([f"• {cond}" for cond in conditions])
        story.append(Paragraph(conditions_html, condition_style))
        story.append(Spacer(1, 0.4*inch))
    
    # 确保有数据
    if not header or not data:
        story.append(Paragraph("CSV文件中没有有效数据", intro_style))
        doc.build(story)
        return
    
    # 创建表格 - 充分利用宽页面
    table_data = [header] + data
    
    # 计算可用内容宽度（页面宽度减去左右边距）
    content_width = CUSTOM_PAGE_SIZE[0] - doc.leftMargin - doc.rightMargin
    
    # 自动计算列宽（更精确的方法）
    col_widths = []
    for col_idx in range(len(header)):
        # 计算该列中最长内容的宽度
        max_len = 0
        for row_idx, row in enumerate(table_data):
            if col_idx < len(row):
                cell_value = str(row[col_idx])
                # 估算宽度：每个字符约6点，加上一些边距
                cell_width = len(cell_value) * 6 + 20
                if cell_width > max_len:
                    max_len = cell_width
        
        # 设置最小和最大列宽
        min_width = 60  # 最小列宽
        max_width = 300  # 最大列宽
        col_width = max(min(max_len, max_width), min_width)
        col_widths.append(col_width)
    
    # 计算总宽度并调整比例
    total_width = sum(col_widths)
    if total_width > content_width:
        # 按比例缩小所有列
        scale_factor = content_width / total_width
        col_widths = [w * scale_factor for w in col_widths]
    else:
        # 如果总宽小于内容宽度，可以添加额外空间到某些列
        extra_space = (content_width - total_width) / len(col_widths)
        col_widths = [w + extra_space for w in col_widths]
    
    # 创建表格
    table = Table(
        table_data, 
        colWidths=col_widths,
        repeatRows=1  # 重复表头
    )
    
    # 应用表格样式
    table_style = TableStyle([
        # 表头样式
        ('BACKGROUND', (0,0), (-1,0), colors.HexColor('#3498DB')),
        ('TEXTCOLOR', (0,0), (-1,0), colors.whitesmoke),
        ('FONTNAME', (0,0), (-1,0), font_name),
        ('FONTSIZE', (0,0), (-1,0), 14),  # 增大字号
        ('BOLD', (0,0), (-1,0), 1),
        ('ALIGN', (0,0), (-1,0), 'CENTER'),
        ('VALIGN', (0,0), (-1,0), 'MIDDLE'),
        
        # 数据行样式
        ('FONTNAME', (0,1), (-1,-1), font_name),
        ('FONTSIZE', (0,1), (-1,-1), 12),  # 增大字号
        ('TEXTCOLOR', (0,1), (-1,-1), colors.HexColor('#2C3E50')),
        ('BOTTOMPADDING', (0,0), (-1,-1), 10),
        ('TOPPADDING', (0,0), (-1,-1), 8),
        ('GRID', (0,0), (-1,-1), 0.8, colors.HexColor('#BDC3C7')),  # 加粗网格线
        
        # 交替行颜色
        ('ROWBACKGROUNDS', (0,1), (-1,-1), 
         [colors.HexColor('#FFFFFF'), colors.HexColor('#F8F9F9')]),
        
        # 最后一行加粗
        ('FONTSIZE', (0,-1), (-1,-1), 13),
        ('BOLD', (0,-1), (-1,-1), 1),
        ('LINEABOVE', (0,-1), (-1,-1), 1.5, colors.HexColor('#E74C3C')),
    ])
    
    # 特殊列对齐（数字列右对齐）
    for i, col in enumerate(header):
        if any(col.lower().find(keyword) != -1 for keyword in ['amount', '金额', '数量', '总计', 'price', 'cost']):
            table_style.add('ALIGN', (i,1), (i,-1), 'RIGHT')
    
    table.setStyle(table_style)
    story.append(table)
    story.append(Spacer(1, 0.4*inch))
    
    # 添加页脚
    footer_text = f"第 <page> 页，共 {len(data)} 条记录，生成时间: {report_date}"
    story.append(Paragraph(footer_text, footer_style))
    
    # 生成PDF
    doc.build(story)

if __name__ == "__main__":
    # 示例用法
    intro_text = """
    本报告基于系统导出的原始交易数据生成，包含详细的交易记录和分析结果。
    报告数据已经过系统自动校验，确保准确性。如需进一步分析或对数据有疑问，
    请联系数据分析部门。
    """
    
    csv_to_pdf(
        input_csv='transaction_report.csv',  # 输入CSV文件
        output_pdf='宽幅交易报告.pdf',        # 输出PDF文件
        title='宽幅交易明细分析报告',         # PDF主标题
        intro_text=intro_text,               # 介绍文本
        image_path='company_banner.png'      # 封面图片路径
    )
```