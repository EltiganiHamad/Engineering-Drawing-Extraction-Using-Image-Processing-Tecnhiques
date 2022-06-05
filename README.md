# Engineering Drawing Extraction Image-Processing Tecnhiques Functions
In this project a python program is developed to identify and extract information and data from a set of engineering drawings with different layouts and designs using a variety of image processing techniques. The methodology used for the extraction of the engineering drawing and the reading of the tabulted data is split in two sections: Image segmentation and data extraction.

## Our Images

Before proceeding on with our methodology it is important to first understand the format of our images. There are a total of 20 Engineering images each with a drawing and tables of information. 

![alt text](https://github.com/EltiganiHamad/Engineering-Drawing-Extraction-Image-Processing-Tecnhiques-Functions-/blob/main/GitHub%20Example%20Image.png)

The area of the image bounded by a blue box covers the engineering drawing where as the green and red boxes are bounded over the engineering data that must be extracted and saved into a csv file. Within the 20 engineering drawings there are six different image arrangmenets each with their own location of the drawings and tables or data. Additionally each of the images with the six different image arrangment have smaller differences in border width, fonts and font sizes. Therefore our methodology has an invidual function for each arrangments to deal with the differences in fonts, borders and font sizes.

## Library Requirements

In order to begin this process, we must first import the required libraries for this task namely OpenCV, numpy, matplotlib, os, openpyxl and pytesseract.

```

import cv2
import numpy as np
import matplotlib.pyplot as plt
import pytesseract
import openpyxl
import os

```


## Image Segmentation:

The first process is to segment any type of engineering drawing into two different images. The drawing image and the table image. The table image is later going to be used to extract the data.

To start the segmentation process, we created a function defined as diagram_segmentation(). Next the image was read using cv2 and converted into grayscale. After applying the OTSU thresholding, the image was inverted which was done for easier detection drawing.

```
def diagram_segmentation(img_path):
    # Read image
    img = cv2.imread(img_path,0)
         
    # Image thresholding
    (thresh, img_bin) = cv2.threshold(img, 120, 255,cv2.THRESH_BINARY|cv2.THRESH_OTSU)
            
    # Converting image to binary
    img_bin = 255-img_bin 
    cv2.imwrite("Image_bin.jpg",img_bin)

```

Next, we defined two different kernels to perform morphological operations. The kernels would be used to detect the vertical and horizontal lines in the drawings with length which was based on the width of the drawing.


```
# Defining a kernel length

kernel_length = np.array(img).shape[1]//50
             
# Verticle kernel to detect all the vertical lines in the image
verticle_kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (1, kernel_length))
# Horizontal kernel to detect all the horizontal lines in the image
hori_kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (kernel_length, 1))
# Set up a (3 x 3) kernel
kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (3, 3))

```
The morphological operators were used to detect the vertical and horizontal lines of the drawings and were stored into two different images.

```

# Detect verticle lines using morphological operations
img_temp1 = cv2.erode(img_bin, verticle_kernel, iterations=3)
verticle_lines_img = cv2.dilate(img_temp1, verticle_kernel, iterations=3)
cv2.imwrite("verticle_lines.jpg",verticle_lines_img)
            
# Detect horizontal lines using morphological operations
img_temp2 = cv2.erode(img_bin, hori_kernel, iterations=3)
horizontal_lines_img = cv2.dilate(img_temp2, hori_kernel, iterations=3)
cv2.imwrite("horizontal_lines.jpg",horizontal_lines_img)

```

After detecting the two images of vertical and horizontal lines, both the images were combined by adding to get the third image using the weighting parameters. The third image gives us the final image of what the drawing part looks like and successfully detects the drawing.

```
# To add into the new image weighting parameters were used
alpha = 0.5
beta = 1.0 - alpha
# Add the two images to get a third image
img_final_bin = cv2.addWeighted(verticle_lines_img, alpha, horizontal_lines_img, beta, 0.0)
img_final_bin = cv2.erode(~img_final_bin, kernel, iterations=2)
(thresh, img_final_bin) = cv2.threshold(img_final_bin, 128, 255, cv2.THRESH_BINARY | cv2.THRESH_OTSU)
            
# Debugging
# The verticle and horizantal lines combination is tested out
cv2.imwrite("img_final_bin.jpg",img_final_bin)

```

Now that we had detected most of the drawing part, we would need to do contouring to extract the actual region of our interest and separate it out of the drawing, so we are left with the table to retrieve information. In order to define the approach of our contouring, another sort_contours function was needed to be defined which would help us even detect the rounded parts of the drawing and able to extract the region of our interest. The first part we defined what we want to sort and later in what direction we needed to sort. The bounding boxes allows us to extract the region of interest.

```

def sort_contours(cnts, method="left-to-right"):
	# initialize sort index
	reverse = False
	i = 0
	# This is if we use the reverse approach
	if method == "right-to-left" or method == "bottom-to-top":
		reverse = True
	# sorting of Y coordinates rather than x coordinates
	if method == "top-to-bottom" or method == "bottom-to-top":
		i = 1
	# sorting top to bottom by constructing bounding boxes
	boundingBoxes = [cv2.boundingRect(c) for c in cnts]
	(cnts, boundingBoxes) = zip(*sorted(zip(cnts, boundingBoxes),
		key=lambda b:b[1][i], reverse=reverse))
	# calling all the list of contours and bounding boxes
	return (cnts, boundingBoxes)
  
 ```
 
 The final image was taken, and we applied contouring with a “top-to-bottom" approach but the contouring only detected and showed us our region of interest. Setting up the parameters and masking them will extract us the drawing.
 
 ```
# Find contours for image, which will detect all the relevent region of interest
contours, hierarchy = cv2.findContours(img_final_bin, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
# Sort all the contours by top to bottom approach
(contours, boundingBoxes) = sort_contours(contours, method="top-to-bottom")

```

Now that we found all the contours, the only thing left was to define the parameters for it to extract, mask, and save it into two different images. In order to extract, we assigned the drawing mask as ones and the table mask as zeros. This will help us in identifying and applying the mask. With the help of the contours, we assigned the parameters of the mask and then applied on the image. The images were saved into two different images and the drawing and the table is successfully extracted. The table image is going to be used further to extract the data and export into excel.

```

# Find the size of the image
[nrow,ncol] = img.shape
# Apply the masks
drawing_mask = np.ones((nrow,ncol)) 
table_mask = np.zeros((nrow,ncol)) 
    for c in contours:
        # To find the location and width and height of the contours
        x, y, w, h = cv2.boundingRect(c)
    
        # If width is between 80 to 1800 and Height between 20 to 1500 except line 286 and 287. Apply mask!
        if (80 < w < 1800 and 20 < h < 1500 ) and w > h and w !=(286) and w !=(287):
            drawing_mask[y:y+h, x:x+w] = 0
            table_mask[y:y+h, x:x+w] = 1
        
drawing = img * drawing_mask
   
tables = img * table_mask
plt.imshow(drawing,cmap='gray')
cv2.imwrite('Drawing.png', drawing)
cv2.imwrite('Tables.png', tables)

```


## Engineering Data Extraction

After performing data segmentation on the image to detect and separate the drawings from the title blocks of the image further processing can be done to extract and export the tabular data from our new segmented table image by creating a new function 'extractTabularData'. 

Then we create a new excel file using the workbook class from the openpyxl library to which all the data extracted from the image will be exported. Then we will read the table image from our directory using Open cv’s imread() function. Im1 is used to detect the contours within the image and we draw those contours onto the untouched image im.

```
file =  r'Tables.png'
    
def extractTabularData1(file):
    
    #Creating a new workshop using the workbook class and titling "AMMENDMENTS"
    wb = openpyxl.Workbook()
    sheet = wb.active
    sheet.title = "AMMENDMENTS"
 

    
    #Reading in the image
    im1 = cv2.imread(file, 0)
    im = cv2.imread(file)


```

Next, we perform a few pre-processing techniques to enhance the details of the image and the data presented in it before contour detection. The first technique we employ is image blurring with a kernel size of 1, this is useful for removing high frequency content such as noise and edges from the image. In addition, we also perform Inverse image thresholding with the thresholds of 100 and 255, which helps to further enhance the image and the data presented within it.


```

    #Bluring the image
    blur = cv2.blur(im1,(1,1))

    #Thresholding the image
    ret,thresh_value = cv2.threshold(blur,100,255,cv2.THRESH_BINARY_INV)

```

Then we set two kernels of size (3,3) to perform image erosion and dilation. The employment of erosion will help to remove small object within the image like noise by diminishing the boundaries of the table within the image. Performing dilation on the image after erosion will help to enlarge back those boundaries and make them more prominent once the noise has been removed therefore making the image details clearer.

```

    #Initializing the kernel for image erosion
    kernel1 = np.ones((3,3), np.uint8)
    
    #Eroding the image
    img_erosion = cv2.erode(thresh_value, kernel1, iterations=1)

    #Initializing the kernel for image dialation
    kernel2 = np.ones((3,3),np.uint8)
    
    #Dialating the image
    dilated_value = cv2.dilate(img_erosion,kernel2,iterations = 1)

```

Now that we have completed all the pre-processing required, we can proceed to find the contours around the image using OpenCV’s findContours function.

```
#Finding the Contours within the image
contours, hierarchy = cv2.findContours(dilated_value,cv2.RETR_TREE,cv2.CHAIN_APPROX_SIMPLE)
```
After the contours have been detected and saved into our contour’s variable, we can start drawing the contours on our image. These contours are drawn on our original image im which has not be manipulated or processed so far. This process is done through a for loop that iterates through every contour bounding it with a rectangle using the cv2.boundingRect function with the dimension x,y,w and h. Once bounded these dimensions are appended into a list called coordinates which is initialized prior to the for loop. At each irritation of the for loop the coordinates of the contours are used along with the colon operator to crop the image and save it into the variable cropped. In addition, a text file ‘recognized ‘ is created inside of the for loop, using the pytesseract.image_to_string() function from the pytesseract library we convert the cropped image to string then write the strings obtained into the text file opened and close the text file. Moreover, within the same for loop once the file is closed, we proceed to exporting the data into excel file.

## Exporting to excel file

In this sector the program opens the recognized.txt in read mode to loop through and capture all the details inside into an excel file. The program then closes the recognized.txt file and draw the contours. This is conditioned using an if statement that only draws the rectangles around the contours only if the y value is less than 50. This condition is important as specifies which area of the image in which the rectangles are drawn. After all the contours are drawn, we show the image with the detected contours using plt.show() and use the cv2.imwrite() to the save the image into our directory.

```

    #For every countour from the variable counters
    for cnt in contours:
        #Bounds a rectangle around the countour
        x,y,w,h = cv2.boundingRect(cnt)
        #appending the cordinates of the rectangle to the list
        cordinates.append((x,y,w,h))
        #Extracting the value of the countor
        cropped = im1[y:y + h, x:x + w] 
        #Opening a text file in append mode
        file = open("recognized.txt", "a")
        #Converting the extracted countour values into string using pytesseract
        text = pytesseract.image_to_string(cropped)
        #Writing the string values into the text files
        file.write(str(text)) 
        file.write("\n")
        #Closing the file
        file.close() 
        #Drawing the countours on our untouched image
        if y > 50:
            cv2.rectangle(im,(x,y),(x+w,y+h),(0,0,255),1)
   
    #Showing and Saving the Detected image
    plt.imshow(im)
    cv2.namedWindow('detecttable', cv2.WINDOW_NORMAL)
    cv2.imwrite('detecttable.jpg',im)


```

Next, we open the text file ‘recognized’ we saved earlier, and read it line by line using readlines() save it into our variable content_list. Then using a for loop, we append each line into the excel sheet we created earlier, this is conditioned using an if statement that filters off any identifiable or understandable text or shapes from the text file. This is an important step as if these are not filtered an error will occur when trying to append them into excel, since excel is unable to understand or identify them as normal string values. After the filter is completed through the for loop the file is closed once again.


```

#Opening the text file    
    f = open("recognized.txt", "r")
    #Reading each line from the text file
    content_list = f.readlines()
    #For each row of text in the variable
    for row in range(len(content_list)):
        #Filtering the unidentifiable text. 
        if(content_list[row] != '\x0c\n'):
         #append the lines to the exel sheet
         sheet.append([content_list[row]])
    #Closing the file
    f.close()

```

The below code is used to remove all the blank lines from the reconized.txt. It identifies the blank lines, removes it and then writes the new results in another file named 'Engineering image details WITHOUT space.txt' and closes the files. This will enable the program to take only the value under a certain header. For example, for image 1, UNIT take ‘mm’.

```

  #Extracting the data from the text file and saving it into another text file without spaces
    f1 = open("recognized.txt", "r")
    for line in f1:
        if not line.isspace():
            file2 = open("Engineering image details WITHOUT space.txt", "a")
            file2.write(line)
            file2.close()
    f1.close()        

```

We have hardcoded the headers to extract their results. To search a result, the program will go through 'Engineering image details WITHOUT space.txt' line by line, looking for the header and saving those lines below it in an excel file as their result. The path of the rows and columns are already coded to arrange each headers and results in the new sheet.

```

    #Converting the file into an object        
    with open("Engineering image details WITHOUT space.txt", 'r') as read_obj:
            #Creating a new sheet
            newsheet =  wb.create_sheet("Extract", 0)
            
            #Searching for the field names in the text file and appending them into the sheet
            for line in read_obj:    

                string_to_search1 = ("UNIT:")
                string_to_search2 = ("DRAWN:")
                string_to_search3 = ("DRAWING TITLE:")
                string_to_search4 = ("CHECKED:")
                string_to_search5 = ("APPROVED:")
                string_to_search6 = ("STATUS:")
                string_to_search7 = ("COMPANY:")
                string_to_search8 = ("PAGE:")
                string_to_search9 = ("DRAWING NUMBER:")



                newsheet['A1'] = (string_to_search1)    
                newsheet['A2'] = (string_to_search2)
                newsheet['A3'] = (string_to_search3)
                newsheet['A4'] = (string_to_search4)
                newsheet['A5'] = (string_to_search5)
                newsheet['A6'] = (string_to_search6)
                newsheet['A7'] = (string_to_search7)
                newsheet['A8'] = (string_to_search8)
                newsheet['A9'] = (string_to_search9)


```

This part of the code shows the path for the results of a header. The program allows to search each header line by line then the next line of the header is coded to column next to it. Once completed, the program saves the workbook then closes the file. We remove both recognized.txt and 'Engineering image details WITHOUT space.txt' to prevent any clashes when iterating for the other images, else it will it keep using the same files as long as the user is using the program. Each time at the start of the program, it creates the files and then deletes them at the end.

```

 #Appending the line below each field name and appending it into the sheet
                if string_to_search1 in line:
                    newsheet['B1'] = next(read_obj)

                if string_to_search2 in line:
                    newsheet['B2'] = next(read_obj)

                if string_to_search3 in line:
                    newsheet['B3'] = next(read_obj)      

                if string_to_search4 in line:
                    newsheet['B4'] = next(read_obj)

                if string_to_search5 in line:
                    newsheet['B5'] = next(read_obj) 

                if string_to_search6 in line:
                    newsheet['B6'] = next(read_obj)

                if string_to_search7 in line:
                    newsheet['B7'] = next(read_obj)

                if string_to_search8 in line:
                    newsheet['B8'] = next(read_obj)

                if string_to_search9 in line:
                    newsheet['B9'] = next(read_obj)

    wb.save("Engineering Data.xlsx")
    os.remove("recognized.txt")
    os.remove("Engineering Data WITHOUT space.txt")
    read_obj.close()

```

We included a menu which consist of a function for each image depending on the structure. When the user input an image number, the program loops and take the function for a certain image.

```

def dataExtraction1(img_path):
    diagram_segmentation(img_path)
    extractTabularData1(file)
    print("DONE")
def dataExtraction2(img_path):
    diagram_segmentation(img_path)
    extractTabularData2(file)
    print("DONE")
def dataExtraction3(img_path):
    diagram_segmentation(img_path)
    extractTabularData3(file)
    print("DONE")
def dataExtractiont4(img_path):
    diagram_segmentation(img_path)
    extractTabularData4(file)
    print("DONE")
def dataExtraction5(img_path):
    diagram_segmentation(img_path)
    extractTabularData5(file)
    print("DONE")
def dataExtraction6(img_path):
    diagram_segmentation(img_path)
    extractTabularData6(file)
    print("DONE")
input = int(input("Enter an image from 1 to 20: "))

while input !=0: 
    if input == 1:    
        dataExtraction1('01.png')
        break
    if input == 2:    
        dataExtraction2('02.png')
        break
    if input == 3:    
        dataExtraction3('03.png')
        break
    if input == 4:    
        dataExtraction4('04.png')
        break
    if input == 5:   
        dataExtraction5('05.png')
        break
    if input == 6:   
        dataExtraction5('06.png')
        break
    if input == 7:     
        dataExtraction5('07.png')
        break
        
    if input == 8:    
        dataExtraction5('08.png')
        break
        
    if input == 9:    
        dataExtraction5('09.png')
        break
        
    if input == 10:    
        dataExtraction5('10.png')
        break
     
    if input == 11:    
        dataExtraction5('11.png')
        break
        
    if input == 12:    
        dataExtraction5('12.png')
        break
        
    if input == 13:    
        dataExtraction5('13.png')
        break
        
    if input == 14:    
        dataExtraction5('14.png')
        break
    
    if input == 15:    
        dataExtraction6('15.png')
        break
        
    if input == 16:    
        dataExtraction6('16.png')
        break
    
    if input == 17:    
        dataExtraction6('17.png')
        break
    
    if input == 18:    
        dataExtraction6('18.png')
        break
        
    if input == 19:    
        dataExtraction6('19.png')
        break
    
    if input == 20:    
        dataExtraction6('20.png')
        break
        
    else:
        print("Invalid input")
        break

```

We have now just explore the entire process of extraction of engineering data and drawings from an image using data extracted through OCR and image segmentation using contour detection. Within the script we develop 5 other functions each with a tailored data extraction function based on the arrangment of the engineering text data within the image. These are called based on the input image number fed into the menu by the user.










    
    




