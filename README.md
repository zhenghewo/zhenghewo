import csv
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image, Table, TableStyle
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import inch

def csv_to_pdf(input_csv, output_pdf, title, intro_text, image_path):
    """
    将CSV文件转换为带有固定图片和文字开头的PDF文件
    
    参数:
    input_csv (str): 输入CSV文件路径
    output_pdf (str): 输出PDF文件路径
    title (str): PDF标题文本
    intro_text (str): 介绍性文本
    image_path (str): 固定图片路径
    """
    
    # 创建PDF文档
    doc = SimpleDocTemplate(
        output_pdf,
        pagesize=letter,
        rightMargin=72,
        leftMargin=72,
        topMargin=72,
        bottomMargin=72
    )
    
    # 获取样式表
    styles = getSampleStyleSheet()
    
    # 自定义样式
    title_style = ParagraphStyle(
        'CustomTitle',
        parent=styles['Heading1'],
        fontSize=18,
        alignment=1,  # 居中
        spaceAfter=12,
        textColor=colors.darkblue
    )
    
    intro_style = ParagraphStyle(
        'CustomIntro',
        parent=styles['BodyText'],
        fontSize=12,
        spaceAfter=18,
        textColor=colors.darkslategray
    )
    
    # 创建内容列表
    story = []
    
    # 添加标题
    story.append(Paragraph(title, title_style))
    
    # 添加图片（调整大小）
    img = Image(image_path, width=2*inch, height=2*inch)
    img.hAlign = 'CENTER'  # 图片居中
    story.append(img)
    story.append(Spacer(1, 0.25*inch))
    
    # 添加介绍文本
    story.append(Paragraph(intro_text, intro_style))
    story.append(Spacer(1, 0.5*inch))
    
    # 读取CSV数据
    data = []
    with open(input_csv, 'r', encoding='utf-8') as csvfile:
        csv_reader = csv.reader(csvfile)
        for row in csv_reader:
            data.append(row)
    
    # 确保有数据
    if not data:
        story.append(Paragraph("CSV文件中没有数据", styles['BodyText']))
        doc.build(story)
        return
    
    # 创建表格
    table = Table(data, repeatRows=1)  # 重复表头
    
    # 应用表格样式
    table_style = TableStyle([
        # 表头样式
        ('BACKGROUND', (0,0), (-1,0), colors.HexColor('#1E90FF')),
        ('TEXTCOLOR', (0,0), (-1,0), colors.whitesmoke),
        ('FONTSIZE', (0,0), (-1,0), 12),
        ('BOLD', (0,0), (-1,0), 1),
        ('ALIGN', (0,0), (-1,0), 'CENTER'),
        
        # 数据行样式
        ('FONT', (0,1), (-1,-1), 'Helvetica'),
        ('FONTSIZE', (0,1), (-1,-1), 10),
        ('BOTTOMPADDING', (0,0), (-1,-1), 6),
        ('BACKGROUND', (0,1), (-1,-1), colors.HexColor('#F5F5F5')),
        ('GRID', (0,0), (-1,-1), 1, colors.lightgrey),
        
        # 交替行颜色
        ('ROWBACKGROUNDS', (0,1), (-1,-1), 
         [colors.white, colors.HexColor('#E8F4FF')])
    ])
    
    table.setStyle(table_style)
    story.append(table)
    
    # 生成PDF
    doc.build(story)

if __name__ == "__main__":
    # 示例用法
    csv_to_pdf(
        input_csv='input_data.csv',        # 输入CSV文件
        output_pdf='output_report.pdf',   # 输出PDF文件
        title='销售数据分析报告',          # PDF标题
        intro_text='本报告展示了2023年公司销售数据的详细分析。数据来源于各部门月度报表，'
                   '经过清洗和验证，包含产品类别、销售额、利润等关键指标。报告生成时间：2023年12月31日。',
        image_path='company_logo.png'     # 公司Logo路径
    )