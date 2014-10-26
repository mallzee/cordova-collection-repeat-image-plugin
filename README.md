# Collection Repeat Image

Making performant scrolling lists in hybrid apps is very difficult. There are a few decent attempts to fix this problem. [IonicFramework Collection Repeat](http://ionicframework.com/docs/api/directive/collectionRepeat/) being the best we've used so far, [Including our own attempt](https://github.com/mallzee/angular-ui-table-view). 

When you do not control the source of the images for these lists you start to get problems when scaling massive images on the main javascript thread. This can lead to crazy jankyness on older phones and newer if the images are massive. 

This plugin offloads the image processing to native land to download the image and scale it to the desired size in a background thread. When everything is ready a base64 string of the image is sent back to javascript land to be injected into the img tag.

It's essentially a wrapper around SDWebImage optimised to work with Ionic's collection repeat

```JavaScript
var rect = image.getBoundingClientRect();

var options = {
  data: src,
  index: scope.id,
  quality: 0,
  scale: Math.round(rect.width) + 'x' + Math.round(rect.height)
};

cordova.plugins.CollectionRepeatImage.getImage(options, function (data) {
    image[0].src = 'data:image/jpeg;base64,' + data;
});
```

## Installation

    cordova plugin add https://github.com/mallzee/cordova-collection-repeat-image-plugin.git

## Supported Platforms

- iOS
- Android (Soon)

## API

- cordova.plugins.CollectionRepeatImage.getImage(options, success, failure)
- cordova.plugins.CollectionRepeatImage.cancel(index)
- cordova.plugins.CollectionRepeatImage.cancelAll()

### options

- url: The url for the specified image
- index: The index associated with this image. Used to cancel jobs when items have gone out of view
- quality: [0 - 1] The compression applied to the image. 
- scale: The image magic format "[width]x[height]" See [here](https://github.com/mustangostang/UIImage-ResizeMagick) for more variations


## Sample Angular Directive

Example of the markup inside of a collection repeat.

```HTML
<div class="product-multi" collection-repeat="product in products track by product.id" collection-item-width="'50%'" collection-item-height="'50%'">
    <mlz-img src="{{product.image}}" id="{{$id}}"></mlz-img>
    <div class="price"{{product.cost}}</div>
</div>
```


Example directive to make use of the plugin.

```JavaScript
angular.directive('mlzImg', [function () {
  return {
    restrict: 'E',
    scope: {
      src: '@',
      id: '@'
    },
    template: '<img />',
    link: function (scope, element) {
      var image = element.children(),
          hasLoader = false,
          hasLoaded = false,
          rect;

      function getDimensions() {
        if (!rect) {
          rect = element[0].getBoundingClientRect();
        }
        return rect;
      }

      image[0].style.display = 'none';

      if (!hasLoader) {
        image.on('load', function () {
          ionic.requestAnimationFrame(function () {
            image[0].style.display = 'block';
          });
          hasLoaded = true;
        }));
        hasLoader = true;
      }

      function getFileExtension(file) {
        var parts = file.split('.');
        return parts[parts.length - 1];
      }

      function replaceWithScaledImage(src, rect) {
        if (ionic.Platform.isWebView()) {
          var options = {
            data: src,
            index: scope.id,
            quality: 0.5,
            scale: Math.round(rect.width) + 'x' + Math.round(rect.height)
          };

          cordova.plugins.CollectionRepeatImage.getImage(options, function (data) {
            image[0].src = 'data:image/jpeg;base64,' + data;
          });
        } else {
          image[0].src = src;
        }
      }

      function loadNewImage(src) {
        ionic.requestAnimationFrame(function () {
          image[0].style.display = 'none';
          replaceWithScaledImage(src, getDimensions());
        });
      }

      scope.$watch('src', function (src, oldSrc) {
        if (src && src !== oldSrc || !hasLoaded) {
          if (ionic.Platform.isWebView()) {
            cordova.plugins.CollectionRepeatImage.cancel(scope.id);
            loadNewImage(src);
          } else {
            loadNewImage(src);
          }
        }
      });
    }
  };
});
```

Cancel all operations on a scope destroy

```JavaScript
$scope.$on('destroy', function () {
  if(ionic.Platform.isWebView()) {
    cordova.plugins.CollectionRepeatImage.cancelAll()
  }
});
```

## Attribution

- [SDWebImage](https://github.com/rs/SDWebImage)
- [UIImage-ResizeMagick](https://github.com/mustangostang/UIImage-ResizeMagick)
